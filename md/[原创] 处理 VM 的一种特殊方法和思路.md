> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-268208.htm)

> [原创] 处理 VM 的一种特殊方法和思路

处理 VM 的一种特殊方法和思路
================

备注：下面是一种处理 VM 的方法，其实还不够完善，栈用数组来模拟的话反编译出来还是不太好看 (虽然比直接撕汇编好)，如果有什么思路大家可以提出建议。

 

整体概述：①先拖入 IDA 分析，得到 "指令" 和 opcode；②通过 python 得到它的汇编；③将 opcode_key 重编为 C 语法 (数组模拟栈，变量模拟寄存器)；④重新将 exe 拖入 IDA 进行分析或调试

 

下面给两个例子：一个是基于栈的 (类似 python 那种)，一个是用函数模拟普通指令的 (有寄存器也有栈)

2020GKCTF-EzMachine
===================

这道题之前讲 VM 的时候讲过，比较经典的一道 VM

[](#一：分析)一：分析
-------------

直接给一个之前的我分析这个 VM 的链接

 

[https://bbs.pediy.com/thread-267670.htm](https://bbs.pediy.com/thread-267670.htm)

[](#二：编写前缀)二：编写前缀
-----------------

准备好 "栈"，push，pop 等，寄存器变量等

 

(根据之前得到的汇编信息来写前缀)

```
#include #include #include void push(int stack[], int* rsp, int data)
{
    *rsp = *rsp + 1;
    stack[*rsp] = data;
}
 
int pop(int stack[], int* rsp)
{
    int ret = stack[*rsp];
    stack[*rsp] = 0;
    *rsp = *rsp-1;
    return ret;
}
 
int main()
{
    int rsp = -1;
    int stack[10] = {0};
    int e_flag;
    int i=0;
    int reg0, reg1, reg2, reg3;
    char* string[5] = {"right", "wrong", "puts", "plz input:", "hacker"};
    char input[0x30];
return 0;
} 
```

三：opcode_key 重编为 C 语法 (数组模拟栈，变量模拟寄存器)
-------------------------------------

opcode_key 变为：

```
opcode_key = {
    0: 'nop',
    1: 'reg{}={};',
    2: 'push(stack, &rsp, {});',
    3: 'push(stack, &rsp, reg{});',
    4: 'reg{}=pop(stack, &rsp);',
    5:  'printf("%s",string[reg3]);',
    6:  'reg{}+=reg{};',
    7:  'reg{}-=reg{};',
    8:  'reg{}*=reg{};',
    9:  'reg{}%=reg{};',
    10: 'reg{}^=reg{};',
    11: 'goto _(3*{}-3);',                     # 3*{}-3
    12: 'e_flag = reg{} - reg{};',
    13: 'if(e_flag) goto _rip+3; else goto _(3*{}-3);',
    14: 'if(e_flag) goto _(3*{}-3); else goto _rip+3;',
    15: 'if(e_flag <= 0) goto _rip+3; else goto _(3*{}-3);',
    16: 'if(e_flag >= 0) goto _rip+3; else goto _(3*{}-3);',
    17: 'scanf("%s",input);',
    18: 'mem_init {} {}',
    19: 'reg{}=pop(stack, &rsp);',
    20: 'reg{}=input[{}];',
    0xff: 'exit(0);'}

```

脚本：

```
# _*_ coding: utf-8 _*_
# editor: SYJ
# function: Reversed By SYJ
# describe:
opcode_team = [0x01, 0x03, 0x03, 0x05, 0x00, 0x00, 0x11, 0x00, 0x00, 0x01, 0x01, 0x11, 0x0C, 0x00, 0x01, 0x0D, 0x0A, 0x00, 0x01, 0x03, 0x01, 0x05, 0x00, 0x00, 0xFF, 0x00, 0x00, 0x01, 0x02, 0x00, 0x01, 0x00, 0x11, 0x0C, 0x00, 0x02, 0x0D, 0x2B, 0x00, 0x14, 0x00, 0x02, 0x01, 0x01, 0x61, 0x0C, 0x00, 0x01, 0x10, 0x1A, 0x00, 0x01, 0x01, 0x7A, 0x0C, 0x00, 0x01, 0x0F, 0x1A, 0x00, 0x01, 0x01, 0x47, 0x0A, 0x00, 0x01, 0x01, 0x01, 0x01, 0x06, 0x00, 0x01, 0x0B, 0x24, 0x00, 0x01, 0x01, 0x41, 0x0C, 0x00, 0x01, 0x10, 0x24, 0x00, 0x01, 0x01, 0x5A, 0x0C, 0x00, 0x01, 0x0F, 0x24, 0x00, 0x01, 0x01, 0x4B, 0x0A, 0x00, 0x01, 0x01, 0x01, 0x01, 0x07, 0x00, 0x01, 0x01, 0x01, 0x10, 0x09, 0x00, 0x01, 0x03, 0x01, 0x00, 0x03, 0x00, 0x00, 0x01, 0x01, 0x01, 0x06, 0x02, 0x01, 0x0B, 0x0B, 0x00, 0x02, 0x07, 0x00, 0x02, 0x0D, 0x00, 0x02, 0x00, 0x00, 0x02, 0x05, 0x00, 0x02, 0x01, 0x00, 0x02, 0x0C, 0x00, 0x02, 0x01, 0x00, 0x02, 0x00, 0x00, 0x02, 0x00, 0x00, 0x02, 0x0D, 0x00, 0x02, 0x05, 0x00, 0x02, 0x0F, 0x00, 0x02, 0x00, 0x00, 0x02, 0x09, 0x00, 0x02, 0x05, 0x00, 0x02, 0x0F, 0x00, 0x02, 0x03, 0x00, 0x02, 0x00, 0x00, 0x02, 0x02, 0x00, 0x02, 0x05, 0x00, 0x02, 0x03, 0x00, 0x02, 0x03, 0x00, 0x02, 0x01, 0x00, 0x02, 0x07, 0x00, 0x02, 0x07, 0x00, 0x02, 0x0B, 0x00, 0x02, 0x02, 0x00, 0x02, 0x01, 0x00, 0x02, 0x02, 0x00, 0x02, 0x07, 0x00, 0x02, 0x02, 0x00, 0x02, 0x0C, 0x00, 0x02, 0x02, 0x00, 0x02, 0x02, 0x00, 0x01, 0x02, 0x01, 0x13, 0x01, 0x02, 0x04, 0x00, 0x00, 0x0C, 0x00, 0x01, 0x0E, 0x5B, 0x00, 0x01, 0x01, 0x22, 0x0C, 0x02, 0x01, 0x0D, 0x59, 0x00, 0x01, 0x01, 0x01, 0x06, 0x02, 0x01, 0x0B, 0x4E, 0x00, 0x01, 0x03, 0x00, 0x05, 0x00, 0x00, 0xFF, 0x00, 0x00, 0x01, 0x03, 0x01, 0x05, 0x00, 0x00, 0xFF, 0x00, 0x00, 0x00]
opcode_key = {
    0: 'nop',
    1: 'reg{}={};',
    2: 'push(stack, &rsp, {});',
    3: 'push(stack, &rsp, reg{});',
    4: 'reg{}=pop(stack, &rsp);',
    5:  'printf("%s",string[reg3]);',
    6:  'reg{}+=reg{};',
    7:  'reg{}-=reg{};',
    8:  'reg{}*=reg{};',
    9:  'reg{}%=reg{};',
    10: 'reg{}^=reg{};',
    11: 'goto _(3*{}-3);',                     # 3*{}-3
    12: 'e_flag = reg{} - reg{};',
    13: 'if(e_flag) goto _rip+3; else goto _(3*{}-3);',
    14: 'if(e_flag) goto _(3*{}-3); else goto _rip+3;',
    15: 'if(e_flag <= 0) goto _rip+3; else goto _(3*{}-3);',
    16: 'if(e_flag >= 0) goto _rip+3; else goto _(3*{}-3);',
    17: 'scanf("%s",input);',
    18: 'mem_init {} {}',
    19: 'reg{}=pop(stack, &rsp);',
    20: 'reg{}=input[{}];',
    0xff: 'exit(0);'}
rip = 0
all = len(opcode_team)
while rip < all:
    x = opcode_team[rip]
    if x != 0:
        print("_" + str(hex(rip)) + ": " + opcode_key[x].format(opcode_team[rip+1], (opcode_team[rip+2])))
        rip += 3
    else:
        print("_" + str(hex(rip)) + ": " + opcode_key[x].format(hex(rip+1)))
        rip += 1

```

最后运行之后得到：

```
_0x0: reg3=3;
_0x3: printf("%s",string[reg3]);
_0x6: scanf("%s",input);
_0x9: reg1=17;
_0xc: e_flag = reg0 - reg1;
_0xf: if(e_flag) goto _rip+3; else goto _(3*10-3);
_0x12: reg3=1;
_0x15: printf("%s",string[reg3]);
_0x18: exit(0);
_0x1b: reg2=0;
_0x1e: reg0=17;
_0x21: e_flag = reg0 - reg2;
_0x24: if(e_flag) goto _rip+3; else goto _(3*43-3);
_0x27: reg0=input[2];
_0x2a: reg1=97;
_0x2d: e_flag = reg0 - reg1;
_0x30: if(e_flag >= 0) goto _rip+3; else goto _(3*26-3);
_0x33: reg1=122;
_0x36: e_flag = reg0 - reg1;
_0x39: if(e_flag <= 0) goto _rip+3; else goto _(3*26-3);
_0x3c: reg1=71;
_0x3f: reg0^=reg1;
_0x42: reg1=1;
_0x45: reg0+=reg1;
_0x48: goto _(3*36-3);
_0x4b: reg1=65;
_0x4e: e_flag = reg0 - reg1;
_0x51: if(e_flag >= 0) goto _rip+3; else goto _(3*36-3);
_0x54: reg1=90;
_0x57: e_flag = reg0 - reg1;
_0x5a: if(e_flag <= 0) goto _rip+3; else goto _(3*36-3);
_0x5d: reg1=75;
_0x60: reg0^=reg1;
_0x63: reg1=1;
_0x66: reg0-=reg1;
_0x69: reg1=16;
_0x6c: reg0%=reg1;
_0x6f: push(stack, &rsp, reg1);
_0x72: push(stack, &rsp, reg0);
_0x75: reg1=1;
_0x78: reg2+=reg1;
_0x7b: goto _(3*11-3);
_0x7e: push(stack, &rsp, 7);
_0x81: push(stack, &rsp, 13);
_0x84: push(stack, &rsp, 0);
_0x87: push(stack, &rsp, 5);
_0x8a: push(stack, &rsp, 1);
_0x8d: push(stack, &rsp, 12);
_0x90: push(stack, &rsp, 1);
_0x93: push(stack, &rsp, 0);
_0x96: push(stack, &rsp, 0);
_0x99: push(stack, &rsp, 13);
_0x9c: push(stack, &rsp, 5);
_0x9f: push(stack, &rsp, 15);
_0xa2: push(stack, &rsp, 0);
_0xa5: push(stack, &rsp, 9);
_0xa8: push(stack, &rsp, 5);
_0xab: push(stack, &rsp, 15);
_0xae: push(stack, &rsp, 3);
_0xb1: push(stack, &rsp, 0);
_0xb4: push(stack, &rsp, 2);
_0xb7: push(stack, &rsp, 5);
_0xba: push(stack, &rsp, 3);
_0xbd: push(stack, &rsp, 3);
_0xc0: push(stack, &rsp, 1);
_0xc3: push(stack, &rsp, 7);
_0xc6: push(stack, &rsp, 7);
_0xc9: push(stack, &rsp, 11);
_0xcc: push(stack, &rsp, 2);
_0xcf: push(stack, &rsp, 1);
_0xd2: push(stack, &rsp, 2);
_0xd5: push(stack, &rsp, 7);
_0xd8: push(stack, &rsp, 2);
_0xdb: push(stack, &rsp, 12);
_0xde: push(stack, &rsp, 2);
_0xe1: push(stack, &rsp, 2);
_0xe4: reg2=1;
_0xe7: reg1=pop(stack, &rsp);
_0xea: reg0=pop(stack, &rsp);
_0xed: e_flag = reg0 - reg1;
_0xf0: if(e_flag) goto _(3*91-3); else goto _rip+3;
_0xf3: reg1=34;
_0xf6: e_flag = reg2 - reg1;
_0xf9: if(e_flag) goto _rip+3; else goto _(3*89-3);
_0xfc: reg1=1;
_0xff: reg2+=reg1;
_0x102: goto _(3*78-3);
_0x105: reg3=0;
_0x108: printf("%s",string[reg3]);
_0x10b: exit(0);
_0x10e: reg3=1;
_0x111: printf("%s",string[reg3]);
_0x114: exit(0);
_0x117: nop

```

手动处理一些细节的地方，粘贴到前缀的后面

```
#include #include #include void push(int stack[], int* rsp, int data)
{
    *rsp = *rsp + 1;
    stack[*rsp] = data;
}
 
int pop(int stack[], int* rsp)
{
    int ret = stack[*rsp];
    stack[*rsp] = 0;
    *rsp = *rsp-1;
    return ret;
}
 
int main()
{
    int rsp = -1;
    int stack[10] = {0};
    int e_flag;
    int i=0;
    int reg0, reg1, reg2, reg3;
    char* string[5] = {"right", "wrong", "puts", "plz input:", "hacker"};
    char input[0x30];
_0x0: reg3=3;
_0x3: printf("%s",string[reg3]);
_0x6: scanf("%s",input);reg0=strlen(input);
_0x9: reg1=17;
_0xc: e_flag = reg0 - reg1;
_0xf: if(e_flag) {goto _0x12;} else {goto _0x1b;}
_0x12: reg3=1;
_0x15: printf("%s",string[reg3]);
_0x18: exit(0);
_0x1b: reg2=0;
_0x1e: reg0=17;
_0x21: e_flag = reg0 - reg2;
_0x24: if(e_flag) {goto _0x27;} else {goto _0x7e;}
_0x27: reg0=input[2];
_0x2a: reg1=97;
_0x2d: e_flag = reg0 - reg1;
_0x30: if(e_flag >= 0) {goto _0x33;} else {goto _0x4b;}
_0x33: reg1=122;
_0x36: e_flag = reg0 - reg1;
_0x39: if(e_flag <= 0) {goto _0x3c;} else {goto _0x4b;}
_0x3c: reg1=71;
_0x3f: reg0^=reg1;
_0x42: reg1=1;
_0x45: reg0+=reg1;
_0x48: goto _0x69;
_0x4b: reg1=65;
_0x4e: e_flag = reg0 - reg1;
_0x51: if(e_flag >= 0) {goto _0x54;} else {goto _0x69;}
_0x54: reg1=90;
_0x57: e_flag = reg0 - reg1;
_0x5a: if(e_flag <= 0) {goto _0x5d;} else {goto _0x69;}
_0x5d: reg1=75;
_0x60: reg0^=reg1;
_0x63: reg1=1;
_0x66: reg0-=reg1;
_0x69: reg1=reg0/16;
_0x6c: reg0%=16;
_0x6f: push(stack, &rsp, reg1);
_0x72: push(stack, &rsp, reg0);
_0x75: reg1=1;
_0x78: reg2+=reg1;
_0x7b: goto _0x1e;
_0x7e: push(stack, &rsp, 7);
_0x81: push(stack, &rsp, 13);
_0x84: push(stack, &rsp, 0);
_0x87: push(stack, &rsp, 5);
_0x8a: push(stack, &rsp, 1);
_0x8d: push(stack, &rsp, 12);
_0x90: push(stack, &rsp, 1);
_0x93: push(stack, &rsp, 0);
_0x96: push(stack, &rsp, 0);
_0x99: push(stack, &rsp, 13);
_0x9c: push(stack, &rsp, 5);
_0x9f: push(stack, &rsp, 15);
_0xa2: push(stack, &rsp, 0);
_0xa5: push(stack, &rsp, 9);
_0xa8: push(stack, &rsp, 5);
_0xab: push(stack, &rsp, 15);
_0xae: push(stack, &rsp, 3);
_0xb1: push(stack, &rsp, 0);
_0xb4: push(stack, &rsp, 2);
_0xb7: push(stack, &rsp, 5);
_0xba: push(stack, &rsp, 3);
_0xbd: push(stack, &rsp, 3);
_0xc0: push(stack, &rsp, 1);
_0xc3: push(stack, &rsp, 7);
_0xc6: push(stack, &rsp, 7);
_0xc9: push(stack, &rsp, 11);
_0xcc: push(stack, &rsp, 2);
_0xcf: push(stack, &rsp, 1);
_0xd2: push(stack, &rsp, 2);
_0xd5: push(stack, &rsp, 7);
_0xd8: push(stack, &rsp, 2);
_0xdb: push(stack, &rsp, 12);
_0xde: push(stack, &rsp, 2);
_0xe1: push(stack, &rsp, 2);
_0xe4: reg2=1;
_0xe7: reg1=pop(stack, &rsp);
_0xea: reg0=pop(stack, &rsp);
_0xed: e_flag = reg0 - reg1;
_0xf0: if(e_flag) {goto _0x10e;} else {goto _0xf3;}
_0xf3: reg1=34;
_0xf6: e_flag = reg2 - reg1;
_0xf9: if(e_flag) {goto _0xfc;} else {goto _0x108;}
_0xfc: reg1=1;
_0xff: reg2+=reg1;
_0x102: goto _0xe7;
_0x105: reg3=0;
_0x108: printf("%s",string[reg3]);
_0x10b: exit(0);
_0x10e: reg3=1;
_0x111: printf("%s",string[reg3]);
_0x114: exit(0);
    return 0;
} 
```

注：，只要编译不报错能得到 exe 就行，有些恢复的不是很完全，就不能完全正确运行，不过这个还勉强

 

![](https://bbs.pediy.com/upload/attach/202106/921830_7BPZKB595SC768E.png)

[](#四：重新将exe拖入ida分析，调试)四：重新将 exe 拖入 IDA 分析，调试
---------------------------------------------

```
int __cdecl __noreturn main(int argc, const char **argv, const char **envp)
{
  int v3; // kr00_4
  char input[48]; // [rsp+20h] [rbp-B0h] BYREF
  char *string[5]; // [rsp+50h] [rbp-80h]
  int stack[10]; // [rsp+80h] [rbp-50h] BYREF
  int rsp_0; // [rsp+B4h] [rbp-1Ch] BYREF
  int e_flag; // [rsp+B8h] [rbp-18h]
  int reg1; // [rsp+BCh] [rbp-14h]
  int reg3; // [rsp+C0h] [rbp-10h]
  int i; // [rsp+C4h] [rbp-Ch]
  int j; // [rsp+C8h] [rbp-8h]
  int reg0; // [rsp+CCh] [rbp-4h]
 
  _main(argc, argv, envp);
  rsp_0 = -1;
  memset(stack, 0, sizeof(stack));
  i = 0;
  string[0] = "right";
  string[1] = "wrong";
  string[2] = "puts";
  string[3] = "plz input:";
  string[4] = "hacker";
  reg3 = 3;
  printf("%s", "plz input:");
  scanf("%s", input);
  reg0 = strlen(input);
  reg1 = 17;
  e_flag = reg0 - 17;
  if ( reg0 != 17 )
  {
    reg3 = 1;
    printf("%s", string[1]);
    exit(0);
  }
  for ( j = 0; ; ++j )
  {
    reg0 = 17;
    e_flag = 17 - j;
    if ( j == 17 )
      break;
    reg0 = input[2];
    reg1 = 97;
    e_flag = input[2] - 97;
    if ( e_flag < 0 || (reg1 = 'z', e_flag = reg0 - 'z', reg0 - 'z' > 0) )
    {
      reg1 = 'A';
      e_flag = reg0 - 'A';
      if ( reg0 - 'A' >= 0 )
      {
        reg1 = 'Z';
        e_flag = reg0 - 'Z';
        if ( reg0 - 'Z' <= 0 )                  // 'A' - 'Z'
        {
          reg0 ^= 75u;
          reg1 = 1;
          --reg0;
        }
      }
    }
    else                                        // 'a' - 'z'
    {
      reg0 ^= 71u;
      reg1 = 1;
      ++reg0;
    }
    v3 = reg0;
    reg1 = reg0 / 16;
    reg0 %= 16;
    push(stack, &rsp_0, v3 / 16);     //push进除16的整数
    push(stack, &rsp_0, reg0);        //push进%16的整数
    reg1 = 1;
  }
  push(stack, &rsp_0, 7);
  push(stack, &rsp_0, 13);
  push(stack, &rsp_0, 0);
  push(stack, &rsp_0, 5);
  push(stack, &rsp_0, 1);
  push(stack, &rsp_0, 12);
  push(stack, &rsp_0, 1);
  push(stack, &rsp_0, 0);
  push(stack, &rsp_0, 0);
  push(stack, &rsp_0, 13);
  push(stack, &rsp_0, 5);
  push(stack, &rsp_0, 15);
  push(stack, &rsp_0, 0);
  push(stack, &rsp_0, 9);
  push(stack, &rsp_0, 5);
  push(stack, &rsp_0, 15);
  push(stack, &rsp_0, 3);
  push(stack, &rsp_0, 0);
  push(stack, &rsp_0, 2);
  push(stack, &rsp_0, 5);
  push(stack, &rsp_0, 3);
  push(stack, &rsp_0, 3);
  push(stack, &rsp_0, 1);
  push(stack, &rsp_0, 7);
  push(stack, &rsp_0, 7);
  push(stack, &rsp_0, 11);
  push(stack, &rsp_0, 2);
  push(stack, &rsp_0, 1);
  push(stack, &rsp_0, 2);
  push(stack, &rsp_0, 7);
  push(stack, &rsp_0, 2);
  push(stack, &rsp_0, 12);
  push(stack, &rsp_0, 2);
  push(stack, &rsp_0, 2);
  for ( j = 1; ; ++j )
  {
    reg1 = pop(stack, &rsp_0);
    reg0 = pop(stack, &rsp_0);
    e_flag = reg0 - reg1;
    if ( reg0 != reg1 )
      break;
    reg1 = 34;
    e_flag = j - 34;
    if ( j == 34 )
    {
      printf("%s", string[reg3]);
      exit(0);
    }
    reg1 = 1;
  }
  reg3 = 1;
  printf("%s", string[1]);
  exit(0);
}

```

分析逻辑后写出解题脚本即可：

```
cmp_data = [0x7,0xd,0x0,0x5,0x1,0xc,0x1,0x0,0x0,0xd,0x5,0xf,0x0,0x9,0x5,0xf,0x3,0x0,0x2,0x5,0x3,0x3,0x1,0x7,0x7,0xb,0x2,0x1,0x2,0x7,0x2,0xc,0x2,0x2,]
cmp_data = cmp_data[::-1]
print(cmp_data)
flag = ''
for i in range(0, 34, 2):
    temp = cmp_data[i] + cmp_data[i+1]*16
    x = ((temp+1) ^ 75)
    y = ((temp-1) ^ 71)
    if (x >= 65) and (x <= 90):    # 'A'-'Z'
        flag += chr(x)
    elif (y >= 97) and (y <= 122):   # 'a' - 'z'
        flag += chr(y)
    else:
        flag += chr(temp)    # 没有处于'a'-'z'或'A'-'Z'之间
print(flag)
# flag{Such_A_EZVM}

```

2020 虎符 CTF-VM
==============

这个是一个基于栈的虚拟机，比较繁杂的一道题目

[](#一：分析)一：分析
-------------

### 1.main 函数以及结构体分析

main 函数中先从我们输入的参数所对应的文件中读取 opcode，然后做一些虚拟机的准备，之后调用一个函数 VM，里面用 while 循环来是实现 dispatch，传入了一个地址 (经过后面分析，这个地址是结构体的首地址)

```
__int64 __fastcall main(int a1, char **a2, char **a3)
{
  void *opcode; // rbx
  FILE *fp; // rbp
  void *v5; // rbx
  char v7[5]; // [rsp+3h] [rbp-25h] BYREF
  unsigned __int64 v8; // [rsp+8h] [rbp-20h]
 
  v8 = __readfsqword(0x28u);
  opcode = malloc(0x400uLL);
  if ( a1 > 1 )
  {
    fp = fopen(a2[1], "rb");
    fread(opcode, 1uLL, 1000uLL, fp);
    fclose(fp);
  }
  code_adr = opcode;
  rip_ = 0;
  rsp_ = 0;
  v5 = malloc(0x100uLL);
  save_local = (__int64)malloc(0x100uLL);
  stack = (__int64)v5;
  v7[0] = 0;
  save_cnt = (__int64)v7;
  v7[1] = 0;
  v7[2] = 0;
  v7[3] = 0;
  v7[4] = 0;
  VM((unsigned int *)&rip_);
  return 0LL;
}

```

在结合了 VM 函数内那些分析之后重命名如下

```
.bss:00000000006020A0 rip_            dd ?                    ; DATA XREF: main+6A↑w
.bss:00000000006020A0                                         ; main+9C↑o
.bss:00000000006020A4 rsp_            dd ?                    ; DATA XREF: main+74↑w
.bss:00000000006020A8 code_adr        dq ?                    ; DATA XREF: main+63↑w
.bss:00000000006020B0 stack           dq ?                    ; DATA XREF: main+A1↑w
.bss:00000000006020B8 save_local      dq ?                    ; DATA XREF: main+90↑w
.bss:00000000006020C0 save_cnt        dq ?                    ; DATA XREF: main+AD↑w

```

结构体如下：

```
struct _VM{
    dword  rip_;
    dword  rsp_;
    qword* code_adr;
    qword* stack;
    qword* save_local;        //存放本地变量的地方,三个数组
    qword* save_cnt;          //存放计数变量的数组的指针
}VMStruct;

```

这里补充一句，IDA 是严格按照指针的性质来反编译的，指针加减偏移的字节，根据它反编译前面的有个 () 里面的来具体而定

### 2. VM 函数分析

在 VM 函数中一些具体的取结构体中的值

```
*(_BYTE *)(code + rip__1)             //根据rip执行对应指令,取操作码
 
 *(code + rip__1 + 1)                  //操作数,单操作码才会出现这个变量
 
VMstruct[1]                            //栈顶rsp,表示栈内有几个数据
 
*VMStruct                              //rip
 
*((_QWORD *)VMStruct + 1);             //qword指针+1说明偏移8字节，取到了code
 
*((_QWORD *)VMStruct + 2)              //偏移2个8字节,偏移16字节取到stack
 
*((_QWORD *)VMStruct + 3)              //取到一片用来存储数据(数组)的空间
 
*((_QWORD *)VMStruct + 4)              //取到储存循环变量的地址

```

主要就是用 while 循环来实现 dispatcher，然后下方就是逐个进行分析了

 

仔细看注释：

```
__int64 __fastcall dispatcher(unsigned int *VMStruct)
{
  __int64 code; // rcx
  __int64 rip_; // rax
  __int64 rip__1; // rdx
  unsigned int v5; // eax
  __int64 stack_5; // rcx
  _BYTE *v7; // rdx
  signed int v8; // eax
  char b; // al
  unsigned int v10; // edx
  __int64 save_cnt_1; // rsi
  _BYTE *v12; // rax
  __int64 v13; // rcx
  char v14; // cl
  int data_1; // er8
  signed int v16; // edx
  __int64 v17; // rdi
  signed int v18; // esi
  int v19; // er8
  signed int v20; // edx
  __int64 v21; // rdi
  signed int v22; // esi
  int data; // er8
  signed int v24; // edx
  __int64 v25; // rdi
  signed int v26; // esi
  signed int v27; // edx
  __int64 v28; // rdi
  signed int v29; // esi
  int data_3; // er8
  signed int v31; // edx
  __int64 v32; // rdi
  signed int v33; // esi
  int data_2; // edi
  signed int v35; // edx
  __int64 v36; // r8
  signed int v37; // esi
  signed int v38; // ecx
  _BYTE *v39; // rax
  __int64 stack_3; // rdx
  signed int v41; // eax
  signed int v42; // esi
  char v43; // cl
  char v44; // cl
  signed int v45; // eax
  signed int v46; // esi
  char v47; // cl
  char v48; // cl
  signed int v49; // eax
  signed int v50; // esi
  char v51; // cl
  char v52; // cl
  signed int v53; // eax
  __int64 v54; // rdx
  signed int v55; // esi
  unsigned __int8 v56; // cl
  _BYTE *v57; // rdx
  signed int v58; // eax
  __int64 v59; // rdx
  signed int v60; // ecx
  unsigned __int8 v61; // si
  signed int v62; // edx
  __int64 v63; // rcx
  signed int v64; // esi
  char v65; // al
  signed int v66; // eax
  char v67; // si
  signed int rip__2; // eax
  signed int v69; // esi
  char v70; // cl
  char v71; // cl
  _BYTE *lo_var; // rdx
  __int64 stack_2; // rcx
  signed int v74; // eax
  __int64 reg1; // rax
  __int64 save_local; // rdx
  char reg0; // cl
  __int64 rsp__1; // rax
  __int64 stack_1; // rdx
  __int64 stack; // rdx
  _IO_FILE *v81; // rsi
  signed int rsp_1; // eax
  char input_char; // al
  __int64 rsp_; // rdx
  __int64 stack_4; // rcx
 
  code = *((_QWORD *)VMStruct + 1);
  rip_ = *VMStruct;
  while ( 2 )                                   // 循环
  {
    rip__1 = (int)rip_;
LABEL_3:
    switch ( *(_BYTE *)(code + rip__1) )
    {
      case 1:                                   // getc,push   rip+=1
        input_char = _IO_getc(stdin);
        rsp_ = (int)VMStruct[1];
        stack_4 = *((_QWORD *)VMStruct + 2);
        VMStruct[1] = rsp_ + 1;
        *(_BYTE *)(stack_4 + rsp_) = input_char;
        code = *((_QWORD *)VMStruct + 1);
        rip_ = *VMStruct + 1;
        *VMStruct = rip_;
        continue;
      case 2:                                   // pop,putc    rip+=1
        stack = *((_QWORD *)VMStruct + 2);
        v81 = stdout;
        rsp_1 = VMStruct[1] - 1;
        VMStruct[1] = rsp_1;
        _IO_putc(*(unsigned __int8 *)(stack + rsp_1), v81);
        code = *((_QWORD *)VMStruct + 1);
        rip_ = *VMStruct + 1;
        *VMStruct = rip_;
        continue;
      case 3:                                   // nop  rip+=1
        rip_ = (unsigned int)(rip_ + 1);
        *VMStruct = rip_;
        continue;
      case 4:                                   // mov reg0, data; push reg0  rip+=2
        reg0 = *(_BYTE *)(code + rip__1 + 1);
        goto push_reg0;
      case 5:                                   // mov reg1, data; mov reg0, *(cnt_adr+reg1); push reg0;     rip += 2
        reg1 = *(unsigned __int8 *)(code + rip__1 + 1);
        save_local = *((_QWORD *)VMStruct + 4);
        goto pop_reg0;
      case 6:                                   // pop *(cnt_adr+data)     rip+=2
        lo_var = (_BYTE *)(*((_QWORD *)VMStruct + 4) + *(unsigned __int8 *)(code + rip__1 + 1));
        goto pop_to_local;
      case 7:                                   // mov reg1, data;  mov reg0, *(save_local + reg1);   push reg0;       rip+=2
        reg1 = *(unsigned __int8 *)(code + rip__1 + 1);
        save_local = *((_QWORD *)VMStruct + 3);
pop_reg0:
        reg0 = *(_BYTE *)(save_local + reg1);
push_reg0:
        rsp__1 = (int)VMStruct[1];
        stack_1 = *((_QWORD *)VMStruct + 2);
        VMStruct[1] = rsp__1 + 1;
        *(_BYTE *)(stack_1 + rsp__1) = reg0;    // push reg0
        code = *((_QWORD *)VMStruct + 1);
        rip_ = *VMStruct + 2;                   // get_next_opcode_index
        *VMStruct = rip_;
        continue;
      case 8:                                   // pop *(save_local+data)    rip+=2
        lo_var = (_BYTE *)(*((_QWORD *)VMStruct + 3) + *(unsigned __int8 *)(code + rip__1 + 1));// get_local_adr
pop_to_local:
        stack_2 = *((_QWORD *)VMStruct + 2);
        v74 = VMStruct[1] - 1;
        VMStruct[1] = v74;
        *lo_var = *(_BYTE *)(stack_2 + v74);
        code = *((_QWORD *)VMStruct + 1);
        rip_ = *VMStruct + 2;
        *VMStruct = rip_;
        continue;
      case 9:                                   // pop rax; pop rbx; push rbx+rax;      rip+= 1
        rip__2 = VMStruct[1];
        stack_3 = *((_QWORD *)VMStruct + 2);
        v69 = rip__2 - 1;
        rip__2 -= 2;
        VMStruct[1] = v69;
        v70 = *(_BYTE *)(stack_3 + v69);
        VMStruct[1] = rip__2;
        v39 = (_BYTE *)(stack_3 + rip__2);
        v71 = *v39 + v70;
        VMStruct[1] = v69;
        LOBYTE(stack_3) = v71;
        goto LABEL_28;
      case 10:                                  // pop rax; pop rbx; push rbx-rax;      rip+= 1
        v66 = VMStruct[1];
        stack_3 = *((_QWORD *)VMStruct + 2);
        v38 = v66 - 1;
        v66 -= 2;
        VMStruct[1] = v38;
        v67 = *(_BYTE *)(stack_3 + v38);
        VMStruct[1] = v66;
        v39 = (_BYTE *)(stack_3 + v66);
        LOBYTE(stack_3) = *v39 - v67;
        goto LABEL_27;
      case 11:                                  // pop rax; pop rbx; push rbx*rax;      rip+= 1
        v62 = VMStruct[1];
        v63 = *((_QWORD *)VMStruct + 2);
        v64 = v62 - 1;
        v62 -= 2;
        VMStruct[1] = v64;
        v65 = *(_BYTE *)(v63 + v64);
        VMStruct[1] = v62;
        v7 = (_BYTE *)(v63 + v62);
        b = *v7 * v65;
        VMStruct[1] = v64;
        goto LABEL_8;
      case 12:                                  // pop rax; pop rbx; push rbx/rax;      rip+= 1
        v58 = VMStruct[1];
        v59 = *((_QWORD *)VMStruct + 2);
        v60 = v58 - 1;
        v58 -= 2;
        VMStruct[1] = v60;
        v61 = *(_BYTE *)(v59 + v60);
        VMStruct[1] = v58;
        v7 = (_BYTE *)(v58 + v59);
        rip_ = (unsigned __int8)*v7;
        if ( !v61 )
          return rip_;
        VMStruct[1] = v60;
        b = (unsigned __int16)rip_ / v61;
        goto LABEL_8;
      case 13:                                  // pop rax; pop rbx; push rbx%rax;      rip+= 1
        v53 = VMStruct[1];
        v54 = *((_QWORD *)VMStruct + 2);
        v55 = v53 - 1;
        v53 -= 2;
        VMStruct[1] = v55;
        v56 = *(_BYTE *)(v54 + v55);
        VMStruct[1] = v53;
        v57 = (_BYTE *)(v53 + v54);
        LOWORD(v53) = (unsigned __int8)*v57;
        VMStruct[1] = v55;
        rip_ = (unsigned __int8)((unsigned __int16)v53 % v56);
        *v57 = rip_;
        if ( !v56 )
          return rip_;
        goto LABEL_9;
      case 14:                                  // pop eax;  pop ebx;  push ebx^eax;      rip+= 1
        v49 = VMStruct[1];
        stack_3 = *((_QWORD *)VMStruct + 2);
        v50 = v49 - 1;
        v49 -= 2;
        VMStruct[1] = v50;
        v51 = *(_BYTE *)(stack_3 + v50);
        VMStruct[1] = v49;
        v39 = (_BYTE *)(stack_3 + v49);
        v52 = *v39 ^ v51;
        VMStruct[1] = v50;
        LOBYTE(stack_3) = v52;
        goto LABEL_28;
      case 15:                                  // pop rax; pop rbx; push rbx&rax;      rip+= 1
        v45 = VMStruct[1];
        stack_3 = *((_QWORD *)VMStruct + 2);
        v46 = v45 - 1;
        v45 -= 2;
        VMStruct[1] = v46;
        v47 = *(_BYTE *)(stack_3 + v46);
        VMStruct[1] = v45;
        v39 = (_BYTE *)(stack_3 + v45);
        v48 = *v39 & v47;
        VMStruct[1] = v46;
        LOBYTE(stack_3) = v48;
        goto LABEL_28;
      case 16:                                  // pop rax; pop rbx; push rbx|rax;      rip+= 1
        v41 = VMStruct[1];
        stack_3 = *((_QWORD *)VMStruct + 2);
        v42 = v41 - 1;
        v41 -= 2;
        VMStruct[1] = v42;
        v43 = *(_BYTE *)(stack_3 + v42);
        VMStruct[1] = v41;
        v39 = (_BYTE *)(stack_3 + v41);
        v44 = *v39 | v43;
        VMStruct[1] = v42;
        LOBYTE(stack_3) = v44;
        goto LABEL_28;
      case 17:                                  // pop reg;  push -reg;      rip+=1
        v38 = VMStruct[1];
        VMStruct[1] = v38 - 1;
        v39 = (_BYTE *)(*((_QWORD *)VMStruct + 2) + v38 - 1);
        LODWORD(stack_3) = -(unsigned __int8)*v39;
        goto LABEL_27;
      case 18:                                  // pop reg; push ~reg;       rip+=1
        v38 = VMStruct[1];
        VMStruct[1] = v38 - 1;
        v39 = (_BYTE *)(*((_QWORD *)VMStruct + 2) + v38 - 1);
        LOBYTE(stack_3) = ~*v39;
LABEL_27:
        VMStruct[1] = v38;
LABEL_28:
        *v39 = stack_3;
        code = *((_QWORD *)VMStruct + 1);
        rip_ = *VMStruct + 1;
        *VMStruct = rip_;
        continue;
      case 19:                                  // pop rax; pop rbx; if stack[rax] != stack[rbx]: rip+=2
                                                // else    rip+=data
        data_1 = *(unsigned __int8 *)(code + rip__1 + 1);
        v27 = VMStruct[1];
        v28 = *((_QWORD *)VMStruct + 2);
        v29 = v27 - 1;
        v27 -= 2;
        VMStruct[1] = v29;
        LOBYTE(v29) = *(_BYTE *)(v28 + v29);
        VMStruct[1] = v27;
        if ( *(_BYTE *)(v28 + v27) != (_BYTE)v29 )
          goto LABEL_21;
        goto LABEL_15;
      case 20:                                  // pop rax; pop rbx; if stack[rax] == stack[rbx]: rip+=2
                                                // else   rip+=data
        data_2 = *(char *)(code + rip__1 + 1);
        v35 = VMStruct[1];
        v36 = *((_QWORD *)VMStruct + 2);
        v37 = v35 - 1;
        v35 -= 2;
        VMStruct[1] = v37;
        LOBYTE(v37) = *(_BYTE *)(v36 + v37);
        VMStruct[1] = v35;
        if ( *(_BYTE *)(v36 + v35) == (_BYTE)v37 )
          goto LABEL_21;
        rip_ = (unsigned int)(data_2 + rip_);
        *VMStruct = rip_;
        continue;
      case 21:                                  // pop rax; pop rbx; if stack[rax] >= stack[rbx]: rip+=2
        data_3 = *(char *)(code + rip__1 + 1);  // else: rip+=data
        v31 = VMStruct[1];
        v32 = *((_QWORD *)VMStruct + 2);
        v33 = v31 - 1;
        v31 -= 2;
        VMStruct[1] = v33;
        LOBYTE(v33) = *(_BYTE *)(v32 + v33);
        VMStruct[1] = v31;
        if ( *(_BYTE *)(v32 + v31) <= (unsigned __int8)v33 )
          goto LABEL_21;
        rip_ = (unsigned int)(data_3 + rip_);
        *VMStruct = rip_;
        continue;
      case 22:                                  // pop rax; pop rbx; if stack[rax] > stack[rbx]: rip+=2
        data = *(char *)(code + rip__1 + 1);    // else: rip+=data
        v24 = VMStruct[1];
        v25 = *((_QWORD *)VMStruct + 2);
        v26 = v24 - 1;
        v24 -= 2;
        VMStruct[1] = v26;
        LOBYTE(v26) = *(_BYTE *)(v25 + v26);
        VMStruct[1] = v24;
        if ( *(_BYTE *)(v25 + v24) < (unsigned __int8)v26 )
          goto LABEL_21;
        rip_ = (unsigned int)(data + rip_);
        *VMStruct = rip_;
        continue;
      case 23:                                  // pop rax; pop rbx; if stack[rax] <= stack[rbx]: rip+=2
        v19 = *(char *)(code + rip__1 + 1);     // else: rip+=data
        v20 = VMStruct[1];
        v21 = *((_QWORD *)VMStruct + 2);
        v22 = v20 - 1;
        v20 -= 2;
        VMStruct[1] = v22;
        LOBYTE(v22) = *(_BYTE *)(v21 + v22);
        VMStruct[1] = v20;
        if ( *(_BYTE *)(v21 + v20) >= (unsigned __int8)v22 )
          goto LABEL_21;
        rip_ = (unsigned int)(v19 + rip_);
        *VMStruct = rip_;
        continue;
      case 24:                                  // pop rax; pop rbx; if stack[rax] < stack[rbx]: rip+=2
        data_1 = *(char *)(code + rip__1 + 1);  // else  rip+=data
        v16 = VMStruct[1];
        v17 = *((_QWORD *)VMStruct + 2);
        v18 = v16 - 1;
        v16 -= 2;
        VMStruct[1] = v18;
        LOBYTE(v18) = *(_BYTE *)(v17 + v18);
        VMStruct[1] = v16;
        if ( *(_BYTE *)(v17 + v16) > (unsigned __int8)v18 )
        {
LABEL_21:
          rip_ = (unsigned int)(rip_ + 2);
          *VMStruct = rip_;
        }
        else
        {
LABEL_15:
          rip_ = (unsigned int)(data_1 + rip_);
          *VMStruct = rip_;
        }
        continue;
      case 25:                                  // pop rax;  push *(save_local+rax)      rip+=1
        v10 = VMStruct[1];
        save_cnt_1 = *((_QWORD *)VMStruct + 3);
        VMStruct[1] = v10 - 1;
        v12 = (_BYTE *)(*((_QWORD *)VMStruct + 2) + (int)(v10 - 1));
        v13 = (unsigned __int8)*v12;
        goto LABEL_11;
      case 26:                                  // pop rax;  pop rbx;  mov *(save_local+rax), rbx
        v5 = VMStruct[1];
        stack_5 = *((_QWORD *)VMStruct + 2);
        VMStruct[1] = v5 - 1;
        v7 = (_BYTE *)(*((_QWORD *)VMStruct + 3) + *(unsigned __int8 *)(stack_5 + (int)(v5 - 1)));
        goto LABEL_7;
      case 27:                                  // pop rax;   push *(save_cnt+rax)        rip+=1
        v10 = VMStruct[1];
        save_cnt_1 = *((_QWORD *)VMStruct + 4);
        VMStruct[1] = v10 - 1;
        v12 = (_BYTE *)(*((_QWORD *)VMStruct + 2) + (int)(v10 - 1));
        v13 = (unsigned __int8)*v12;
LABEL_11:
        v14 = *(_BYTE *)(save_cnt_1 + v13);
        VMStruct[1] = v10;
        *v12 = v14;
        code = *((_QWORD *)VMStruct + 1);
        rip_ = *VMStruct + 1;
        *VMStruct = rip_;
        continue;
      case 28:                                  // pop rax; pop rbx; mov *(save_cnt+rax), rbx;   rip+=1
        v5 = VMStruct[1];
        stack_5 = *((_QWORD *)VMStruct + 2);
        VMStruct[1] = v5 - 1;
        v7 = (_BYTE *)(*((_QWORD *)VMStruct + 4) + *(unsigned __int8 *)(stack_5 + (int)(v5 - 1)));
LABEL_7:
        v8 = v5 - 2;
        VMStruct[1] = v8;
        b = *(_BYTE *)(stack_5 + v8);
LABEL_8:
        *v7 = b;
LABEL_9:
        code = *((_QWORD *)VMStruct + 1);
        rip_ = *VMStruct + 1;
        *VMStruct = rip_;
        continue;
      case 29:                                  // if *(code+data) <= 29: while_continue
                                                // else:  return
                                                // (这个功能就是判断是否执行到了最后,因为给的opcode最后一个就是30)
        rip_ = (unsigned int)(*(char *)(code + rip__1 + 1) + (_DWORD)rip_);
        rip__1 = (int)rip_;
        *VMStruct = rip_;
        if ( *(_BYTE *)(code + (int)rip_) <= 29u )
          goto LABEL_3;
        return rip_;
      default:
        return rip_;
    }
  }
}

```

可以整理得到初步的 opcode_key(这个 vm 的指令集)：

 

一点也没简化过的，用来对照 IDA 进行理解

```
opcode_key = {
    1: "getc,push",
    2: "pop,putc",
    3: "nop",
    4: "mov reg0,{};  push reg0",
    5: "mov reg1,{};  mov reg0,*(cnt_adr+reg1);  push reg0",
    6: "pop *(cnt_adr+{})",
    7: "mov reg1,{};  mov reg0,*(save_cnt + reg1);  push reg0",
    8: "pop *(save_local+{})",
    9: "pop rax;  pop rbx;  push rbx+rax",
    10: "pop rax;  pop rbx;  push rbx-rax",
    11: "pop rax;  pop rbx;  push rbx*rax",
    12: "pop rax;  pop rbx;  push rbx/rax",
    13: "pop rax;  pop rbx;  push rbx%rax",
    14: "pop eax;  pop ebx;  push ebx^eax",
    15: "pop rax;  pop rbx;  push rbx&rax",
    16: "pop rax;  pop rbx;  push rbx|rax",
    17: "pop reg;  push (-reg)",
    18: "pop reg;  push ~(reg)",
    19: "pop rax;  pop rbx;  if stack[rax]!=stack[rbx]:rip+=2  else:rip+={}",
    20: "pop rax;  pop rbx;  if stack[rax]==stack[rbx]:rip+=2  else:rip+={}",
    21: "pop rax;  pop rbx;  if stack[rax]>=stack[rbx]:rip+=2  else:rip+={}",
    22: "pop rax;  pop rbx;  if stack[rax]>stack[rbx]:rip+=2  else:rip+={}",
    23: "pop rax;  pop rbx;  if stack[rax]<=stack[rbx]:rip+=2  else:rip+={}",
    24: "pop rax;  pop rbx;  if stack[rax]
```

理解指令并简化：

```
opcode_key = {
    1: "push getc",
    2: "putc pop",
    3: "nop",
    4: "push {}",
    5: "push *(cnt_adr+{})",
    6: "pop *(cnt_adr+{})",
    7: "push *(save_local + {})",
    8: "pop *(save_local+{})",
    9: "pop rax;  pop rbx;  push rbx+rax",
    10: "pop rax;  pop rbx;  push rbx-rax",
    11: "pop rax;  pop rbx;  push rbx*rax",
    12: "pop rax;  pop rbx;  push rbx/rax",
    13: "pop rax;  pop rbx;  push rbx%rax",
    14: "pop eax;  pop ebx;  push ebx^eax",
    15: "pop rax;  pop rbx;  push rbx&rax",
    16: "pop rax;  pop rbx;  push rbx|rax",
    17: "pop reg;  push (-reg)",
    18: "pop reg;  push ~(reg)",
    19: "pop rax;  pop rbx;  if stack[rax]!=stack[rbx]:rip+=2  else:rip+={}",
    20: "pop rax;  pop rbx;  if stack[rax]==stack[rbx]:rip+=2  else:rip+={}",
    21: "pop rax;  pop rbx;  if stack[rax]>=stack[rbx]:rip+=2  else:rip+={}",
    22: "pop rax;  pop rbx;  if stack[rax]>stack[rbx]:rip+=2  else:rip+={}",
    23: "pop rax;  pop rbx;  if stack[rax]<=stack[rbx]:rip+=2  else:rip+={}",
    24: "pop rax;  pop rbx;  if stack[rax]
```

[](#二：写出初步的脚本得到汇编)二：写出初步的脚本得到汇编
-------------------------------

```
# _*_ coding: utf-8 _*_
# editor: SYJ
# function: Reversed By SYJ
# describe:
"""
# first_part: 给一个数组赋值
code_index = 0          # x在opcode_team中的下标
while code_index <= 0xA6:
    x = opcode_team[code_index]
    if x in opcode_key:
        print(opcode_key[x].format(hex(opcode_team[code_index+1])))
        code_index += 2
# print(hex(code_index))
[0x66, 0x4E, 0xA9, 0xFD, 0x3C, 0x55, 0x90, 0x24, 0x57, 0xF6, 0x5D, 0xB1, 0x01, 0x20, 0x81, 0xFD, 0x36, 0xA9, 0x1F, 0xA1, 0x0E, 0x0D, 0x80, 0x8F, 0xCE, 0x77, 0xE8, 0x23, 0x9E, 0x27, 0x60, 0x2F, 0xA5, 0xCF, 0x1B, 0xBD, 0x32, 0xDB, 0xFF, 0x28, 0xA4, 0x5D]
 
# second_part
while code_index <= 0x124:
    x = opcode_team[code_index]
    if x == 1:
        print(opcode_key[x])
        code_index += 1
    elif x == 8:
        print(opcode_key[x].format(hex(opcode_team[code_index+1])))
        code_index += 2
# print(hex(code_index))
"""
import ctypes
opcode_team = [0x04, 0x66, 0x08, 0x32, 0x04, 0x4E, 0x08, 0x33, 0x04, 0xA9, 0x08, 0x34, 0x04, 0xFD, 0x08, 0x35, 0x04, 0x3C, 0x08, 0x36, 0x04, 0x55, 0x08, 0x37, 0x04, 0x90, 0x08, 0x38, 0x04, 0x24, 0x08, 0x39, 0x04, 0x57, 0x08, 0x3A, 0x04, 0xF6, 0x08, 0x3B, 0x04, 0x5D, 0x08, 0x3C, 0x04, 0xB1, 0x08, 0x3D, 0x04, 0x01, 0x08, 0x3E, 0x04, 0x20, 0x08, 0x3F, 0x04, 0x81, 0x08, 0x40, 0x04, 0xFD, 0x08, 0x41, 0x04, 0x36, 0x08, 0x42, 0x04, 0xA9, 0x08, 0x43, 0x04, 0x1F, 0x08, 0x44, 0x04, 0xA1, 0x08, 0x45, 0x04, 0x0E, 0x08, 0x46, 0x04, 0x0D, 0x08, 0x47, 0x04, 0x80, 0x08, 0x48, 0x04, 0x8F, 0x08, 0x49, 0x04, 0xCE, 0x08, 0x4A, 0x04, 0x77, 0x08, 0x4B, 0x04, 0xE8, 0x08, 0x4C, 0x04, 0x23, 0x08, 0x4D, 0x04, 0x9E, 0x08, 0x4E, 0x04, 0x27, 0x08, 0x4F, 0x04, 0x60, 0x08, 0x50, 0x04, 0x2F, 0x08, 0x51, 0x04, 0xA5, 0x08, 0x52, 0x04, 0xCF, 0x08, 0x53, 0x04, 0x1B, 0x08, 0x54, 0x04, 0xBD, 0x08, 0x55, 0x04, 0x32, 0x08, 0x56, 0x04, 0xDB, 0x08, 0x57, 0x04, 0xFF, 0x08, 0x58, 0x04, 0x28, 0x08, 0x59, 0x04, 0xA4, 0x08, 0x5A, 0x04, 0x5D, 0x08, 0x5B, 0x01, 0x08, 0x64, 0x01, 0x08, 0x65, 0x01, 0x08, 0x66, 0x01, 0x08, 0x67, 0x01, 0x08, 0x68, 0x01, 0x08, 0x69, 0x01, 0x08, 0x6A, 0x01, 0x08, 0x6B, 0x01, 0x08, 0x6C, 0x01, 0x08, 0x6D, 0x01, 0x08, 0x6E, 0x01, 0x08, 0x6F, 0x01, 0x08, 0x70, 0x01, 0x08, 0x71, 0x01, 0x08, 0x72, 0x01, 0x08, 0x73, 0x01, 0x08, 0x74, 0x01, 0x08, 0x75, 0x01, 0x08, 0x76, 0x01, 0x08, 0x77, 0x01, 0x08, 0x78, 0x01, 0x08, 0x79, 0x01, 0x08, 0x7A, 0x01, 0x08, 0x7B, 0x01, 0x08, 0x7C, 0x01, 0x08, 0x7D, 0x01, 0x08, 0x7E, 0x01, 0x08, 0x7F, 0x01, 0x08, 0x80, 0x01, 0x08, 0x81, 0x01, 0x08, 0x82, 0x01, 0x08, 0x83, 0x01, 0x08, 0x84, 0x01, 0x08, 0x85, 0x01, 0x08, 0x86, 0x01, 0x08, 0x87, 0x01, 0x08, 0x88, 0x01, 0x08, 0x89, 0x01, 0x08, 0x8A, 0x01, 0x08, 0x8B, 0x01, 0x08, 0x8C, 0x01, 0x08, 0x8D, 0x04, 0x00, 0x06, 0x00, 0x05, 0x00, 0x04, 0x07, 0x16, 0x56, 0x04, 0x00, 0x06, 0x01, 0x05, 0x01, 0x04, 0x06, 0x16, 0x42, 0x05, 0x00, 0x04, 0x06, 0x0B, 0x05, 0x01, 0x09, 0x04, 0x64, 0x09, 0x19, 0x12, 0x05, 0x00, 0x05, 0x01, 0x04, 0x02, 0x09, 0x0B, 0x0F, 0x04, 0x64, 0x05, 0x00, 0x04, 0x06, 0x0B, 0x05, 0x01, 0x09, 0x09, 0x19, 0x05, 0x00, 0x05, 0x01, 0x04, 0x02, 0x09, 0x0B, 0x12, 0x0F, 0x10, 0x05, 0x01, 0x04, 0x07, 0x0B, 0x05, 0x00, 0x09, 0x1A, 0x05, 0x01, 0x04, 0x01, 0x09, 0x04, 0x01, 0x1C, 0x1D, 0xBC, 0x05, 0x00, 0x04, 0x01, 0x09, 0x04, 0x00, 0x1C, 0x1D, 0xA8, 0x04, 0x01, 0x06, 0x00, 0x05, 0x00, 0x04, 0x2A, 0x16, 0x34, 0x05, 0x00, 0x04, 0x02, 0x0D, 0x04, 0x00, 0x14, 0x0F, 0x05, 0x00, 0x19, 0x05, 0x00, 0x04, 0x01, 0x0A, 0x19, 0x09, 0x05, 0x00, 0x1A, 0x05, 0x00, 0x04, 0x02, 0x0D, 0x04, 0x01, 0x14, 0x0B, 0x04, 0x6B, 0x05, 0x00, 0x19, 0x0B, 0x05, 0x00, 0x1A, 0x05, 0x00, 0x04, 0x01, 0x09, 0x04, 0x00, 0x1C, 0x1D, 0xCA, 0x04, 0x00, 0x06, 0x00, 0x05, 0x00, 0x04, 0x29, 0x18, 0x04, 0x1D, 0x1B, 0x05, 0x00, 0x19, 0x04, 0x32, 0x05, 0x00, 0x09, 0x19, 0x14, 0x0C, 0x05, 0x00, 0x04, 0x01, 0x09, 0x04, 0x00, 0x1C, 0x1D, 0xE5, 0x04, 0x6E, 0x02, 0x1E, 0x04, 0x79, 0x02, 0x1E]
opcode_key = {
    1: "push getc",
    2: "putc pop",
    3: "nop",
    4: "push {}",
    5: "push *(cnt_adr+{})",
    6: "pop *(cnt_adr+{})",
    7: "push *(save_local + {})",
    8: "pop *(save_local+{})",
    9: "pop rax;  pop rbx;  push rbx+rax",
    10: "pop rax;  pop rbx;  push rbx-rax",
    11: "pop rax;  pop rbx;  push rbx*rax",
    12: "pop rax;  pop rbx;  push rbx/rax",
    13: "pop rax;  pop rbx;  push rbx%rax",
    14: "pop eax;  pop ebx;  push ebx^eax",
    15: "pop rax;  pop rbx;  push rbx&rax",
    16: "pop rax;  pop rbx;  push rbx|rax",
    17: "pop reg;  push (-reg)",
    18: "pop reg;  push ~(reg)",
    19: "pop rax;  pop rbx;  if stack[rax]!=stack[rbx]:rip+=2  else:rip+={}",
    20: "pop rax;  pop rbx;  if stack[rax]==stack[rbx]:rip+=2  else:rip+={}",
    21: "pop rax;  pop rbx;  if stack[rax]>=stack[rbx]:rip+=2  else:rip+={}",
    22: "pop rax;  pop rbx;  if stack[rax]>stack[rbx]:rip+=2  else:rip+={}",
    23: "pop rax;  pop rbx;  if stack[rax]<=stack[rbx]:rip+=2  else:rip+={}",
    24: "pop rax;  pop rbx;  if stack[rax]
```

运行之后得到它的汇编

```
_0x0: push 0x66
_0x2: pop *(save_local+0x32)
_0x4: push 0x4e
_0x6: pop *(save_local+0x33)
_0x8: push 0xa9
_0xa: pop *(save_local+0x34)
_0xc: push 0xfd
_0xe: pop *(save_local+0x35)
_0x10: push 0x3c
_0x12: pop *(save_local+0x36)
_0x14: push 0x55
_0x16: pop *(save_local+0x37)
_0x18: push 0x90
_0x1a: pop *(save_local+0x38)
_0x1c: push 0x24
_0x1e: pop *(save_local+0x39)
_0x20: push 0x57
_0x22: pop *(save_local+0x3a)
_0x24: push 0xf6
_0x26: pop *(save_local+0x3b)
_0x28: push 0x5d
_0x2a: pop *(save_local+0x3c)
_0x2c: push 0xb1
_0x2e: pop *(save_local+0x3d)
_0x30: push 0x1
_0x32: pop *(save_local+0x3e)
_0x34: push 0x20
_0x36: pop *(save_local+0x3f)
_0x38: push 0x81
_0x3a: pop *(save_local+0x40)
_0x3c: push 0xfd
_0x3e: pop *(save_local+0x41)
_0x40: push 0x36
_0x42: pop *(save_local+0x42)
_0x44: push 0xa9
_0x46: pop *(save_local+0x43)
_0x48: push 0x1f
_0x4a: pop *(save_local+0x44)
_0x4c: push 0xa1
_0x4e: pop *(save_local+0x45)
_0x50: push 0xe
_0x52: pop *(save_local+0x46)
_0x54: push 0xd
_0x56: pop *(save_local+0x47)
_0x58: push 0x80
_0x5a: pop *(save_local+0x48)
_0x5c: push 0x8f
_0x5e: pop *(save_local+0x49)
_0x60: push 0xce
_0x62: pop *(save_local+0x4a)
_0x64: push 0x77
_0x66: pop *(save_local+0x4b)
_0x68: push 0xe8
_0x6a: pop *(save_local+0x4c)
_0x6c: push 0x23
_0x6e: pop *(save_local+0x4d)
_0x70: push 0x9e
_0x72: pop *(save_local+0x4e)
_0x74: push 0x27
_0x76: pop *(save_local+0x4f)
_0x78: push 0x60
_0x7a: pop *(save_local+0x50)
_0x7c: push 0x2f
_0x7e: pop *(save_local+0x51)
_0x80: push 0xa5
_0x82: pop *(save_local+0x52)
_0x84: push 0xcf
_0x86: pop *(save_local+0x53)
_0x88: push 0x1b
_0x8a: pop *(save_local+0x54)
_0x8c: push 0xbd
_0x8e: pop *(save_local+0x55)
_0x90: push 0x32
_0x92: pop *(save_local+0x56)
_0x94: push 0xdb
_0x96: pop *(save_local+0x57)
_0x98: push 0xff
_0x9a: pop *(save_local+0x58)
_0x9c: push 0x28
_0x9e: pop *(save_local+0x59)
_0xa0: push 0xa4
_0xa2: pop *(save_local+0x5a)
_0xa4: push 0x5d
_0xa6: pop *(save_local+0x5b)
_0xa8: push getc
_0xa9: pop *(save_local+0x64)
_0xab: push getc
_0xac: pop *(save_local+0x65)
_0xae: push getc
_0xaf: pop *(save_local+0x66)
_0xb1: push getc
_0xb2: pop *(save_local+0x67)
_0xb4: push getc
_0xb5: pop *(save_local+0x68)
_0xb7: push getc
_0xb8: pop *(save_local+0x69)
_0xba: push getc
_0xbb: pop *(save_local+0x6a)
_0xbd: push getc
_0xbe: pop *(save_local+0x6b)
_0xc0: push getc
_0xc1: pop *(save_local+0x6c)
_0xc3: push getc
_0xc4: pop *(save_local+0x6d)
_0xc6: push getc
_0xc7: pop *(save_local+0x6e)
_0xc9: push getc
_0xca: pop *(save_local+0x6f)
_0xcc: push getc
_0xcd: pop *(save_local+0x70)
_0xcf: push getc
_0xd0: pop *(save_local+0x71)
_0xd2: push getc
_0xd3: pop *(save_local+0x72)
_0xd5: push getc
_0xd6: pop *(save_local+0x73)
_0xd8: push getc
_0xd9: pop *(save_local+0x74)
_0xdb: push getc
_0xdc: pop *(save_local+0x75)
_0xde: push getc
_0xdf: pop *(save_local+0x76)
_0xe1: push getc
_0xe2: pop *(save_local+0x77)
_0xe4: push getc
_0xe5: pop *(save_local+0x78)
_0xe7: push getc
_0xe8: pop *(save_local+0x79)
_0xea: push getc
_0xeb: pop *(save_local+0x7a)
_0xed: push getc
_0xee: pop *(save_local+0x7b)
_0xf0: push getc
_0xf1: pop *(save_local+0x7c)
_0xf3: push getc
_0xf4: pop *(save_local+0x7d)
_0xf6: push getc
_0xf7: pop *(save_local+0x7e)
_0xf9: push getc
_0xfa: pop *(save_local+0x7f)
_0xfc: push getc
_0xfd: pop *(save_local+0x80)
_0xff: push getc
_0x100: pop *(save_local+0x81)
_0x102: push getc
_0x103: pop *(save_local+0x82)
_0x105: push getc
_0x106: pop *(save_local+0x83)
_0x108: push getc
_0x109: pop *(save_local+0x84)
_0x10b: push getc
_0x10c: pop *(save_local+0x85)
_0x10e: push getc
_0x10f: pop *(save_local+0x86)
_0x111: push getc
_0x112: pop *(save_local+0x87)
_0x114: push getc
_0x115: pop *(save_local+0x88)
_0x117: push getc
_0x118: pop *(save_local+0x89)
_0x11a: push getc
_0x11b: pop *(save_local+0x8a)
_0x11d: push getc
_0x11e: pop *(save_local+0x8b)
_0x120: push getc
_0x121: pop *(save_local+0x8c)
_0x123: push getc
_0x124: pop *(save_local+0x8d)
_0x126: push 0x0
_0x128: pop *(cnt_adr+0x0)
_0x12a: push *(cnt_adr+0x0)
_0x12c: push 0x7
_0x12e: pop rax;  pop rbx;  if stack[rax]>stack[rbx]:rip+=2  else:rip+=0x130
_0x130: push 0x0
_0x132: pop *(cnt_adr+0x1)
_0x134: push *(cnt_adr+0x1)
_0x136: push 0x6
_0x138: pop rax;  pop rbx;  if stack[rax]>stack[rbx]:rip+=2  else:rip+=0x13a
_0x13a: push *(cnt_adr+0x0)
_0x13c: push 0x6
_0x13e: pop rax;  pop rbx;  push rbx*rax
_0x13f: push *(cnt_adr+0x1)
_0x141: pop rax;  pop rbx;  push rbx+rax
_0x142: push 0x64
_0x144: pop rax;  pop rbx;  push rbx+rax
_0x145: pop rax;  push *(save_local+rax)
_0x146: pop reg;  push ~(reg)
_0x147: push *(cnt_adr+0x0)
_0x149: push *(cnt_adr+0x1)
_0x14b: push 0x2
_0x14d: pop rax;  pop rbx;  push rbx+rax
_0x14e: pop rax;  pop rbx;  push rbx*rax
_0x14f: pop rax;  pop rbx;  push rbx&rax
_0x150: push 0x64
_0x152: push *(cnt_adr+0x0)
_0x154: push 0x6
_0x156: pop rax;  pop rbx;  push rbx*rax
_0x157: push *(cnt_adr+0x1)
_0x159: pop rax;  pop rbx;  push rbx+rax
_0x15a: pop rax;  pop rbx;  push rbx+rax
_0x15b: pop rax;  push *(save_local+rax)
_0x15c: push *(cnt_adr+0x0)
_0x15e: push *(cnt_adr+0x1)
_0x160: push 0x2
_0x162: pop rax;  pop rbx;  push rbx+rax
_0x163: pop rax;  pop rbx;  push rbx*rax
_0x164: pop reg;  push ~(reg)
_0x165: pop rax;  pop rbx;  push rbx&rax
_0x166: pop rax;  pop rbx;  push rbx|rax
_0x167: push *(cnt_adr+0x1)
_0x169: push 0x7
_0x16b: pop rax;  pop rbx;  push rbx*rax
_0x16c: push *(cnt_adr+0x0)
_0x16e: pop rax;  pop rbx;  push rbx+rax
_0x16f: pop rax;  pop rbx;  mov *(save_local+rax), rbx
_0x170: push *(cnt_adr+0x1)
_0x172: push 0x1
_0x174: pop rax;  pop rbx;  push rbx+rax
_0x175: push 0x1
_0x177: pop rax;  pop rbx; mov *(save_cnt+rax), rbx
_0x178: jmp 0x134
_0x17a: push *(cnt_adr+0x0)
_0x17c: push 0x1
_0x17e: pop rax;  pop rbx;  push rbx+rax
_0x17f: push 0x0
_0x181: pop rax;  pop rbx; mov *(save_cnt+rax), rbx
_0x182: jmp 0x12a
_0x184: push 0x1
_0x186: pop *(cnt_adr+0x0)
_0x188: push *(cnt_adr+0x0)
_0x18a: push 0x2a
_0x18c: pop rax;  pop rbx;  if stack[rax]>stack[rbx]:rip+=2  else:rip+=0x18e
_0x18e: push *(cnt_adr+0x0)
_0x190: push 0x2
_0x192: pop rax;  pop rbx;  push rbx%rax
_0x193: push 0x0
_0x195: pop rax;  pop rbx;  if stack[rax]==stack[rbx]:rip+=2  else:rip+=0x197
_0x197: push *(cnt_adr+0x0)
_0x199: pop rax;  push *(save_local+rax)
_0x19a: push *(cnt_adr+0x0)
_0x19c: push 0x1
_0x19e: pop rax;  pop rbx;  push rbx-rax
_0x19f: pop rax;  push *(save_local+rax)
_0x1a0: pop rax;  pop rbx;  push rbx+rax
_0x1a1: push *(cnt_adr+0x0)
_0x1a3: pop rax;  pop rbx;  mov *(save_local+rax), rbx
_0x1a4: push *(cnt_adr+0x0)
_0x1a6: push 0x2
_0x1a8: pop rax;  pop rbx;  push rbx%rax
_0x1a9: push 0x1
_0x1ab: pop rax;  pop rbx;  if stack[rax]==stack[rbx]:rip+=2  else:rip+=0x1ad
_0x1ad: push 0x6b
_0x1af: push *(cnt_adr+0x0)
_0x1b1: pop rax;  push *(save_local+rax)
_0x1b2: pop rax;  pop rbx;  push rbx*rax
_0x1b3: push *(cnt_adr+0x0)
_0x1b5: pop rax;  pop rbx;  mov *(save_local+rax), rbx
_0x1b6: push *(cnt_adr+0x0)
_0x1b8: push 0x1
_0x1ba: pop rax;  pop rbx;  push rbx+rax
_0x1bb: push 0x0
_0x1bd: pop rax;  pop rbx; mov *(save_cnt+rax), rbx
_0x1be: jmp 0x188
_0x1c0: push 0x0
_0x1c2: pop *(cnt_adr+0x0)
_0x1c4: push *(cnt_adr+0x0)
_0x1c6: push 0x29
_0x1c8: pop rax;  pop rbx;  if stack[rax]
```

[](#三：分析汇编后写出c重编程序的前缀)三：分析汇编后写出 C 重编程序的前缀
-----------------------------------------

```
#include #include int main()
{
int rsp = -1;            //描述栈顶的位置
char temp;              //定义一个temp变量备用
int stack[66] = {0};    //实现栈
int cnt[0x100] = {0};   //储存计数器的地址
char mem[0x300] = {0};  //储存数据的那一块地址
int rax;
int rbx;
int reg;
return 0;
} 
```

[](#四：opcode_key重编为c语法)四：opcode_key 重编为 C 语法
--------------------------------------------

如下：

```
opcode_key = {
    1: "temp=getchar();rsp+=1;stack[rsp]=temp;",
    2: "putchar(stack[rsp]);rsp-=1;",
    3: "nop;",
    4: "rsp+=1;stack[rsp]={};",
    5: "rsp+=1;stack[rsp]=cnt[{}];",
    6: "cnt[{}]=stack[rsp];rsp-=1;",
    7: "rsp+=1;stack[rsp]=mem[{}];",
    8: "mem[{}]=stack[rsp];rsp-=1;",
    9: "rax=stack[rsp];rsp-=1;rbx=stack[rsp];rsp-=1;rsp+=1;stack[rsp]=rbx+rax;",
    10: "rax=stack[rsp];rsp-=1;rbx=stack[rsp];rsp-=1;rsp+=1;stack[rsp]=rbx-rax;",
    11: "rax=stack[rsp];rsp-=1;rbx=stack[rsp];rsp-=1;rsp+=1;stack[rsp]=rbx*rax;",
    12: "rax=stack[rsp];rsp-=1;rbx=stack[rsp];rsp-=1;rsp+=1;stack[rsp]=rbx/rax;",
    13: "rax=stack[rsp];rsp-=1;rbx=stack[rsp];rsp-=1;rsp+=1;stack[rsp]=rbx%rax;",
    14: "rax=stack[rsp];rsp-=1;rbx=stack[rsp];rsp-=1;rsp+=1;stack[rsp]=rbx^rax;",
    15: "rax=stack[rsp];rsp-=1;rbx=stack[rsp];rsp-=1;rsp+=1;stack[rsp]=rbx&rax;",
    16: "rax=stack[rsp];rsp-=1;rbx=stack[rsp];rsp-=1;rsp+=1;stack[rsp]=rbx|rax;",
    17: "reg=stack[rsp];rsp-=1;rsp+=1;stack[rsp]=(-reg);",
    18: "reg=stack[rsp];rsp-=1;rsp+=1;stack[rsp]=~(reg);",
    19: "rax=stack[rsp];rsp-=1;rbx=stack[rsp];rsp-=1;if (stack[rax]!=stack[rbx]) goto _{};  else goto _{};",   # 这几条指令要重写
    20: "rax=stack[rsp];rsp-=1;rbx=stack[rsp];rsp-=1;if (stack[rax]==stack[rbx]) goto _{};  else goto _{};",
    21: "rax=stack[rsp];rsp-=1;rbx=stack[rsp];rsp-=1;if (stack[rax]>=stack[rbx]) goto _{};  else goto _{};",
    22: "rax=stack[rsp];rsp-=1;rbx=stack[rsp];rsp-=1;if (stack[rax]>stack[rbx]) goto _{};  else goto _{};",
    23: "rax=stack[rsp];rsp-=1;rbx=stack[rsp];rsp-=1;if (stack[rax]<=stack[rbx]) goto _{}; else goto _{};",
    24: "rax=stack[rsp];rsp-=1;rbx=stack[rsp];rsp-=1;if (stack[rax]
```

[](#五：脚本的其它不变，将opcode_key替换之后运行得到)五：脚本的其它不变，将 opcode_key 替换之后运行得到
-----------------------------------------------------------------

```
_0x0: rsp+=1;stack[rsp]=0x66;
_0x2: mem[0x32]=stack[rsp];rsp-=1;
_0x4: rsp+=1;stack[rsp]=0x4e;
_0x6: mem[0x33]=stack[rsp];rsp-=1;
_0x8: rsp+=1;stack[rsp]=0xa9;
_0xa: mem[0x34]=stack[rsp];rsp-=1;
_0xc: rsp+=1;stack[rsp]=0xfd;
_0xe: mem[0x35]=stack[rsp];rsp-=1;
_0x10: rsp+=1;stack[rsp]=0x3c;
_0x12: mem[0x36]=stack[rsp];rsp-=1;
_0x14: rsp+=1;stack[rsp]=0x55;
_0x16: mem[0x37]=stack[rsp];rsp-=1;
_0x18: rsp+=1;stack[rsp]=0x90;
_0x1a: mem[0x38]=stack[rsp];rsp-=1;
_0x1c: rsp+=1;stack[rsp]=0x24;
_0x1e: mem[0x39]=stack[rsp];rsp-=1;
_0x20: rsp+=1;stack[rsp]=0x57;
_0x22: mem[0x3a]=stack[rsp];rsp-=1;
_0x24: rsp+=1;stack[rsp]=0xf6;
_0x26: mem[0x3b]=stack[rsp];rsp-=1;
_0x28: rsp+=1;stack[rsp]=0x5d;
_0x2a: mem[0x3c]=stack[rsp];rsp-=1;
_0x2c: rsp+=1;stack[rsp]=0xb1;
_0x2e: mem[0x3d]=stack[rsp];rsp-=1;
_0x30: rsp+=1;stack[rsp]=0x1;
_0x32: mem[0x3e]=stack[rsp];rsp-=1;
_0x34: rsp+=1;stack[rsp]=0x20;
_0x36: mem[0x3f]=stack[rsp];rsp-=1;
_0x38: rsp+=1;stack[rsp]=0x81;
_0x3a: mem[0x40]=stack[rsp];rsp-=1;
_0x3c: rsp+=1;stack[rsp]=0xfd;
_0x3e: mem[0x41]=stack[rsp];rsp-=1;
_0x40: rsp+=1;stack[rsp]=0x36;
_0x42: mem[0x42]=stack[rsp];rsp-=1;
_0x44: rsp+=1;stack[rsp]=0xa9;
_0x46: mem[0x43]=stack[rsp];rsp-=1;
_0x48: rsp+=1;stack[rsp]=0x1f;
_0x4a: mem[0x44]=stack[rsp];rsp-=1;
_0x4c: rsp+=1;stack[rsp]=0xa1;
_0x4e: mem[0x45]=stack[rsp];rsp-=1;
_0x50: rsp+=1;stack[rsp]=0xe;
_0x52: mem[0x46]=stack[rsp];rsp-=1;
_0x54: rsp+=1;stack[rsp]=0xd;
_0x56: mem[0x47]=stack[rsp];rsp-=1;
_0x58: rsp+=1;stack[rsp]=0x80;
_0x5a: mem[0x48]=stack[rsp];rsp-=1;
_0x5c: rsp+=1;stack[rsp]=0x8f;
_0x5e: mem[0x49]=stack[rsp];rsp-=1;
_0x60: rsp+=1;stack[rsp]=0xce;
_0x62: mem[0x4a]=stack[rsp];rsp-=1;
_0x64: rsp+=1;stack[rsp]=0x77;
_0x66: mem[0x4b]=stack[rsp];rsp-=1;
_0x68: rsp+=1;stack[rsp]=0xe8;
_0x6a: mem[0x4c]=stack[rsp];rsp-=1;
_0x6c: rsp+=1;stack[rsp]=0x23;
_0x6e: mem[0x4d]=stack[rsp];rsp-=1;
_0x70: rsp+=1;stack[rsp]=0x9e;
_0x72: mem[0x4e]=stack[rsp];rsp-=1;
_0x74: rsp+=1;stack[rsp]=0x27;
_0x76: mem[0x4f]=stack[rsp];rsp-=1;
_0x78: rsp+=1;stack[rsp]=0x60;
_0x7a: mem[0x50]=stack[rsp];rsp-=1;
_0x7c: rsp+=1;stack[rsp]=0x2f;
_0x7e: mem[0x51]=stack[rsp];rsp-=1;
_0x80: rsp+=1;stack[rsp]=0xa5;
_0x82: mem[0x52]=stack[rsp];rsp-=1;
_0x84: rsp+=1;stack[rsp]=0xcf;
_0x86: mem[0x53]=stack[rsp];rsp-=1;
_0x88: rsp+=1;stack[rsp]=0x1b;
_0x8a: mem[0x54]=stack[rsp];rsp-=1;
_0x8c: rsp+=1;stack[rsp]=0xbd;
_0x8e: mem[0x55]=stack[rsp];rsp-=1;
_0x90: rsp+=1;stack[rsp]=0x32;
_0x92: mem[0x56]=stack[rsp];rsp-=1;
_0x94: rsp+=1;stack[rsp]=0xdb;
_0x96: mem[0x57]=stack[rsp];rsp-=1;
_0x98: rsp+=1;stack[rsp]=0xff;
_0x9a: mem[0x58]=stack[rsp];rsp-=1;
_0x9c: rsp+=1;stack[rsp]=0x28;
_0x9e: mem[0x59]=stack[rsp];rsp-=1;
_0xa0: rsp+=1;stack[rsp]=0xa4;
_0xa2: mem[0x5a]=stack[rsp];rsp-=1;
_0xa4: rsp+=1;stack[rsp]=0x5d;
_0xa6: mem[0x5b]=stack[rsp];rsp-=1;
_0xa8: temp=getchar();rsp+=1;stack[rsp]=temp;
_0xa9: mem[0x64]=stack[rsp];rsp-=1;
_0xab: temp=getchar();rsp+=1;stack[rsp]=temp;
_0xac: mem[0x65]=stack[rsp];rsp-=1;
_0xae: temp=getchar();rsp+=1;stack[rsp]=temp;
_0xaf: mem[0x66]=stack[rsp];rsp-=1;
_0xb1: temp=getchar();rsp+=1;stack[rsp]=temp;
_0xb2: mem[0x67]=stack[rsp];rsp-=1;
_0xb4: temp=getchar();rsp+=1;stack[rsp]=temp;
_0xb5: mem[0x68]=stack[rsp];rsp-=1;
_0xb7: temp=getchar();rsp+=1;stack[rsp]=temp;
_0xb8: mem[0x69]=stack[rsp];rsp-=1;
_0xba: temp=getchar();rsp+=1;stack[rsp]=temp;
_0xbb: mem[0x6a]=stack[rsp];rsp-=1;
_0xbd: temp=getchar();rsp+=1;stack[rsp]=temp;
_0xbe: mem[0x6b]=stack[rsp];rsp-=1;
_0xc0: temp=getchar();rsp+=1;stack[rsp]=temp;
_0xc1: mem[0x6c]=stack[rsp];rsp-=1;
_0xc3: temp=getchar();rsp+=1;stack[rsp]=temp;
_0xc4: mem[0x6d]=stack[rsp];rsp-=1;
_0xc6: temp=getchar();rsp+=1;stack[rsp]=temp;
_0xc7: mem[0x6e]=stack[rsp];rsp-=1;
_0xc9: temp=getchar();rsp+=1;stack[rsp]=temp;
_0xca: mem[0x6f]=stack[rsp];rsp-=1;
_0xcc: temp=getchar();rsp+=1;stack[rsp]=temp;
_0xcd: mem[0x70]=stack[rsp];rsp-=1;
_0xcf: temp=getchar();rsp+=1;stack[rsp]=temp;
_0xd0: mem[0x71]=stack[rsp];rsp-=1;
_0xd2: temp=getchar();rsp+=1;stack[rsp]=temp;
_0xd3: mem[0x72]=stack[rsp];rsp-=1;
_0xd5: temp=getchar();rsp+=1;stack[rsp]=temp;
_0xd6: mem[0x73]=stack[rsp];rsp-=1;
_0xd8: temp=getchar();rsp+=1;stack[rsp]=temp;
_0xd9: mem[0x74]=stack[rsp];rsp-=1;
_0xdb: temp=getchar();rsp+=1;stack[rsp]=temp;
_0xdc: mem[0x75]=stack[rsp];rsp-=1;
_0xde: temp=getchar();rsp+=1;stack[rsp]=temp;
_0xdf: mem[0x76]=stack[rsp];rsp-=1;
_0xe1: temp=getchar();rsp+=1;stack[rsp]=temp;
_0xe2: mem[0x77]=stack[rsp];rsp-=1;
_0xe4: temp=getchar();rsp+=1;stack[rsp]=temp;
_0xe5: mem[0x78]=stack[rsp];rsp-=1;
_0xe7: temp=getchar();rsp+=1;stack[rsp]=temp;
_0xe8: mem[0x79]=stack[rsp];rsp-=1;
_0xea: temp=getchar();rsp+=1;stack[rsp]=temp;
_0xeb: mem[0x7a]=stack[rsp];rsp-=1;
_0xed: temp=getchar();rsp+=1;stack[rsp]=temp;
_0xee: mem[0x7b]=stack[rsp];rsp-=1;
_0xf0: temp=getchar();rsp+=1;stack[rsp]=temp;
_0xf1: mem[0x7c]=stack[rsp];rsp-=1;
_0xf3: temp=getchar();rsp+=1;stack[rsp]=temp;
_0xf4: mem[0x7d]=stack[rsp];rsp-=1;
_0xf6: temp=getchar();rsp+=1;stack[rsp]=temp;
_0xf7: mem[0x7e]=stack[rsp];rsp-=1;
_0xf9: temp=getchar();rsp+=1;stack[rsp]=temp;
_0xfa: mem[0x7f]=stack[rsp];rsp-=1;
_0xfc: temp=getchar();rsp+=1;stack[rsp]=temp;
_0xfd: mem[0x80]=stack[rsp];rsp-=1;
_0xff: temp=getchar();rsp+=1;stack[rsp]=temp;
_0x100: mem[0x81]=stack[rsp];rsp-=1;
_0x102: temp=getchar();rsp+=1;stack[rsp]=temp;
_0x103: mem[0x82]=stack[rsp];rsp-=1;
_0x105: temp=getchar();rsp+=1;stack[rsp]=temp;
_0x106: mem[0x83]=stack[rsp];rsp-=1;
_0x108: temp=getchar();rsp+=1;stack[rsp]=temp;
_0x109: mem[0x84]=stack[rsp];rsp-=1;
_0x10b: temp=getchar();rsp+=1;stack[rsp]=temp;
_0x10c: mem[0x85]=stack[rsp];rsp-=1;
_0x10e: temp=getchar();rsp+=1;stack[rsp]=temp;
_0x10f: mem[0x86]=stack[rsp];rsp-=1;
_0x111: temp=getchar();rsp+=1;stack[rsp]=temp;
_0x112: mem[0x87]=stack[rsp];rsp-=1;
_0x114: temp=getchar();rsp+=1;stack[rsp]=temp;
_0x115: mem[0x88]=stack[rsp];rsp-=1;
_0x117: temp=getchar();rsp+=1;stack[rsp]=temp;
_0x118: mem[0x89]=stack[rsp];rsp-=1;
_0x11a: temp=getchar();rsp+=1;stack[rsp]=temp;
_0x11b: mem[0x8a]=stack[rsp];rsp-=1;
_0x11d: temp=getchar();rsp+=1;stack[rsp]=temp;
_0x11e: mem[0x8b]=stack[rsp];rsp-=1;
_0x120: temp=getchar();rsp+=1;stack[rsp]=temp;
_0x121: mem[0x8c]=stack[rsp];rsp-=1;
_0x123: temp=getchar();rsp+=1;stack[rsp]=temp;
_0x124: mem[0x8d]=stack[rsp];rsp-=1;
_0x126: rsp+=1;stack[rsp]=0x0;
_0x128: cnt[0x0]=stack[rsp];rsp-=1;
_0x12a: rsp+=1;stack[rsp]=cnt[0x0];
_0x12c: rsp+=1;stack[rsp]=0x7;
_0x12e: rax=stack[rsp];rsp-=1;rbx=stack[rsp];rsp-=1;if (stack[rax]>stack[rbx]) goto _0x130;  else goto _0x184;
_0x130: rsp+=1;stack[rsp]=0x0;
_0x132: cnt[0x1]=stack[rsp];rsp-=1;
_0x134: rsp+=1;stack[rsp]=cnt[0x1];
_0x136: rsp+=1;stack[rsp]=0x6;
_0x138: rax=stack[rsp];rsp-=1;rbx=stack[rsp];rsp-=1;if (stack[rax]>stack[rbx]) goto _0x13a;  else goto _0x17a;
_0x13a: rsp+=1;stack[rsp]=cnt[0x0];
_0x13c: rsp+=1;stack[rsp]=0x6;
_0x13e: rax=stack[rsp];rsp-=1;rbx=stack[rsp];rsp-=1;rsp+=1;stack[rsp]=rbx*rax;
_0x13f: rsp+=1;stack[rsp]=cnt[0x1];
_0x141: rax=stack[rsp];rsp-=1;rbx=stack[rsp];rsp-=1;rsp+=1;stack[rsp]=rbx+rax;
_0x142: rsp+=1;stack[rsp]=0x64;
_0x144: rax=stack[rsp];rsp-=1;rbx=stack[rsp];rsp-=1;rsp+=1;stack[rsp]=rbx+rax;
_0x145: rax=stack[rsp];rsp-=1;rsp+=1;stack[rsp]=mem[rax];
_0x146: reg=stack[rsp];rsp-=1;rsp+=1;stack[rsp]=~(reg);
_0x147: rsp+=1;stack[rsp]=cnt[0x0];
_0x149: rsp+=1;stack[rsp]=cnt[0x1];
_0x14b: rsp+=1;stack[rsp]=0x2;
_0x14d: rax=stack[rsp];rsp-=1;rbx=stack[rsp];rsp-=1;rsp+=1;stack[rsp]=rbx+rax;
_0x14e: rax=stack[rsp];rsp-=1;rbx=stack[rsp];rsp-=1;rsp+=1;stack[rsp]=rbx*rax;
_0x14f: rax=stack[rsp];rsp-=1;rbx=stack[rsp];rsp-=1;rsp+=1;stack[rsp]=rbx&rax;
_0x150: rsp+=1;stack[rsp]=0x64;
_0x152: rsp+=1;stack[rsp]=cnt[0x0];
_0x154: rsp+=1;stack[rsp]=0x6;
_0x156: rax=stack[rsp];rsp-=1;rbx=stack[rsp];rsp-=1;rsp+=1;stack[rsp]=rbx*rax;
_0x157: rsp+=1;stack[rsp]=cnt[0x1];
_0x159: rax=stack[rsp];rsp-=1;rbx=stack[rsp];rsp-=1;rsp+=1;stack[rsp]=rbx+rax;
_0x15a: rax=stack[rsp];rsp-=1;rbx=stack[rsp];rsp-=1;rsp+=1;stack[rsp]=rbx+rax;
_0x15b: rax=stack[rsp];rsp-=1;rsp+=1;stack[rsp]=mem[rax];
_0x15c: rsp+=1;stack[rsp]=cnt[0x0];
_0x15e: rsp+=1;stack[rsp]=cnt[0x1];
_0x160: rsp+=1;stack[rsp]=0x2;
_0x162: rax=stack[rsp];rsp-=1;rbx=stack[rsp];rsp-=1;rsp+=1;stack[rsp]=rbx+rax;
_0x163: rax=stack[rsp];rsp-=1;rbx=stack[rsp];rsp-=1;rsp+=1;stack[rsp]=rbx*rax;
_0x164: reg=stack[rsp];rsp-=1;rsp+=1;stack[rsp]=~(reg);
_0x165: rax=stack[rsp];rsp-=1;rbx=stack[rsp];rsp-=1;rsp+=1;stack[rsp]=rbx&rax;
_0x166: rax=stack[rsp];rsp-=1;rbx=stack[rsp];rsp-=1;rsp+=1;stack[rsp]=rbx|rax;
_0x167: rsp+=1;stack[rsp]=cnt[0x1];
_0x169: rsp+=1;stack[rsp]=0x7;
_0x16b: rax=stack[rsp];rsp-=1;rbx=stack[rsp];rsp-=1;rsp+=1;stack[rsp]=rbx*rax;
_0x16c: rsp+=1;stack[rsp]=cnt[0x0];
_0x16e: rax=stack[rsp];rsp-=1;rbx=stack[rsp];rsp-=1;rsp+=1;stack[rsp]=rbx+rax;
_0x16f: rax=stack[rsp];rsp-=1;rbx=stack[rsp];rsp-=1;mem[rax]=rbx;
_0x170: rsp+=1;stack[rsp]=cnt[0x1];
_0x172: rsp+=1;stack[rsp]=0x1;
_0x174: rax=stack[rsp];rsp-=1;rbx=stack[rsp];rsp-=1;rsp+=1;stack[rsp]=rbx+rax;
_0x175: rsp+=1;stack[rsp]=0x1;
_0x177: rax=stack[rsp];rsp-=1;rbx=stack[rsp];rsp-=1;cnt[rax]=rbx;
_0x178: goto _0x134;
_0x17a: rsp+=1;stack[rsp]=cnt[0x0];
_0x17c: rsp+=1;stack[rsp]=0x1;
_0x17e: rax=stack[rsp];rsp-=1;rbx=stack[rsp];rsp-=1;rsp+=1;stack[rsp]=rbx+rax;
_0x17f: rsp+=1;stack[rsp]=0x0;
_0x181: rax=stack[rsp];rsp-=1;rbx=stack[rsp];rsp-=1;cnt[rax]=rbx;
_0x182: goto _0x12a;
_0x184: rsp+=1;stack[rsp]=0x1;
_0x186: cnt[0x0]=stack[rsp];rsp-=1;
_0x188: rsp+=1;stack[rsp]=cnt[0x0];
_0x18a: rsp+=1;stack[rsp]=0x2a;
_0x18c: rax=stack[rsp];rsp-=1;rbx=stack[rsp];rsp-=1;if (stack[rax]>stack[rbx]) goto _0x18e;  else goto _0x1c0;
_0x18e: rsp+=1;stack[rsp]=cnt[0x0];
_0x190: rsp+=1;stack[rsp]=0x2;
_0x192: rax=stack[rsp];rsp-=1;rbx=stack[rsp];rsp-=1;rsp+=1;stack[rsp]=rbx%rax;
_0x193: rsp+=1;stack[rsp]=0x0;
_0x195: rax=stack[rsp];rsp-=1;rbx=stack[rsp];rsp-=1;if (stack[rax]==stack[rbx]) goto _0x197;  else goto _0x1a4;
_0x197: rsp+=1;stack[rsp]=cnt[0x0];
_0x199: rax=stack[rsp];rsp-=1;rsp+=1;stack[rsp]=mem[rax];
_0x19a: rsp+=1;stack[rsp]=cnt[0x0];
_0x19c: rsp+=1;stack[rsp]=0x1;
_0x19e: rax=stack[rsp];rsp-=1;rbx=stack[rsp];rsp-=1;rsp+=1;stack[rsp]=rbx-rax;
_0x19f: rax=stack[rsp];rsp-=1;rsp+=1;stack[rsp]=mem[rax];
_0x1a0: rax=stack[rsp];rsp-=1;rbx=stack[rsp];rsp-=1;rsp+=1;stack[rsp]=rbx+rax;
_0x1a1: rsp+=1;stack[rsp]=cnt[0x0];
_0x1a3: rax=stack[rsp];rsp-=1;rbx=stack[rsp];rsp-=1;mem[rax]=rbx;
_0x1a4: rsp+=1;stack[rsp]=cnt[0x0];
_0x1a6: rsp+=1;stack[rsp]=0x2;
_0x1a8: rax=stack[rsp];rsp-=1;rbx=stack[rsp];rsp-=1;rsp+=1;stack[rsp]=rbx%rax;
_0x1a9: rsp+=1;stack[rsp]=0x1;
_0x1ab: rax=stack[rsp];rsp-=1;rbx=stack[rsp];rsp-=1;if (stack[rax]==stack[rbx]) goto _0x1ad;  else goto _0x1b6;
_0x1ad: rsp+=1;stack[rsp]=0x6b;
_0x1af: rsp+=1;stack[rsp]=cnt[0x0];
_0x1b1: rax=stack[rsp];rsp-=1;rsp+=1;stack[rsp]=mem[rax];
_0x1b2: rax=stack[rsp];rsp-=1;rbx=stack[rsp];rsp-=1;rsp+=1;stack[rsp]=rbx*rax;
_0x1b3: rsp+=1;stack[rsp]=cnt[0x0];
_0x1b5: rax=stack[rsp];rsp-=1;rbx=stack[rsp];rsp-=1;mem[rax]=rbx;
_0x1b6: rsp+=1;stack[rsp]=cnt[0x0];
_0x1b8: rsp+=1;stack[rsp]=0x1;
_0x1ba: rax=stack[rsp];rsp-=1;rbx=stack[rsp];rsp-=1;rsp+=1;stack[rsp]=rbx+rax;
_0x1bb: rsp+=1;stack[rsp]=0x0;
_0x1bd: rax=stack[rsp];rsp-=1;rbx=stack[rsp];rsp-=1;cnt[rax]=rbx;
_0x1be: goto _0x188;
_0x1c0: rsp+=1;stack[rsp]=0x0;
_0x1c2: cnt[0x0]=stack[rsp];rsp-=1;
_0x1c4: rsp+=1;stack[rsp]=cnt[0x0];
_0x1c6: rsp+=1;stack[rsp]=0x29;
_0x1c8: rax=stack[rsp];rsp-=1;rbx=stack[rsp];rsp-=1;if (stack[rax]
```

手动处理一些细节的地方，粘贴到前缀的后面：

```
#include 
#include 
 
int main()
{
int rsp = -1;            //描述栈顶的位置
char temp;              //定义一个temp变量备用
int stack[66] = {0};    //实现栈
int cnt[0x100] = {0};   //储存计数器的地址
char mem[0x300] = {0};  //储存数据的那一块地址
int rax;
int rbx;
int reg;
_0x0: rsp+=1;stack[rsp]=0x66;
_0x2: mem[0x32]=stack[rsp];rsp-=1;
_0x4: rsp+=1;stack[rsp]=0x4e;
_0x6: mem[0x33]=stack[rsp];rsp-=1;
_0x8: rsp+=1;stack[rsp]=0xa9;
_0xa: mem[0x34]=stack[rsp];rsp-=1;
_0xc: rsp+=1;stack[rsp]=0xfd;
_0xe: mem[0x35]=stack[rsp];rsp-=1;
_0x10: rsp+=1;stack[rsp]=0x3c;
_0x12: mem[0x36]=stack[rsp];rsp-=1;
_0x14: rsp+=1;stack[rsp]=0x55;
_0x16: mem[0x37]=stack[rsp];rsp-=1;
_0x18: rsp+=1;stack[rsp]=0x90;
_0x1a: mem[0x38]=stack[rsp];rsp-=1;
_0x1c: rsp+=1;stack[rsp]=0x24;
_0x1e: mem[0x39]=stack[rsp];rsp-=1;
_0x20: rsp+=1;stack[rsp]=0x57;
_0x22: mem[0x3a]=stack[rsp];rsp-=1;
_0x24: rsp+=1;stack[rsp]=0xf6;
_0x26: mem[0x3b]=stack[rsp];rsp-=1;
_0x28: rsp+=1;stack[rsp]=0x5d;
_0x2a: mem[0x3c]=stack[rsp];rsp-=1;
_0x2c: rsp+=1;stack[rsp]=0xb1;
_0x2e: mem[0x3d]=stack[rsp];rsp-=1;
_0x30: rsp+=1;stack[rsp]=0x1;
_0x32: mem[0x3e]=stack[rsp];rsp-=1;
_0x34: rsp+=1;stack[rsp]=0x20;
_0x36: mem[0x3f]=stack[rsp];rsp-=1;
_0x38: rsp+=1;stack[rsp]=0x81;
_0x3a: mem[0x40]=stack[rsp];rsp-=1;
_0x3c: rsp+=1;stack[rsp]=0xfd;
_0x3e: mem[0x41]=stack[rsp];rsp-=1;
_0x40: rsp+=1;stack[rsp]=0x36;
_0x42: mem[0x42]=stack[rsp];rsp-=1;
_0x44: rsp+=1;stack[rsp]=0xa9;
_0x46: mem[0x43]=stack[rsp];rsp-=1;
_0x48: rsp+=1;stack[rsp]=0x1f;
_0x4a: mem[0x44]=stack[rsp];rsp-=1;
_0x4c: rsp+=1;stack[rsp]=0xa1;
_0x4e: mem[0x45]=stack[rsp];rsp-=1;
_0x50: rsp+=1;stack[rsp]=0xe;
_0x52: mem[0x46]=stack[rsp];rsp-=1;
_0x54: rsp+=1;stack[rsp]=0xd;
_0x56: mem[0x47]=stack[rsp];rsp-=1;
_0x58: rsp+=1;stack[rsp]=0x80;
_0x5a: mem[0x48]=stack[rsp];rsp-=1;
_0x5c: rsp+=1;stack[rsp]=0x8f;
_0x5e: mem[0x49]=stack[rsp];rsp-=1;
_0x60: rsp+=1;stack[rsp]=0xce;
_0x62: mem[0x4a]=stack[rsp];rsp-=1;
_0x64: rsp+=1;stack[rsp]=0x77;
_0x66: mem[0x4b]=stack[rsp];rsp-=1;
_0x68: rsp+=1;stack[rsp]=0xe8;
_0x6a: mem[0x4c]=stack[rsp];rsp-=1;
_0x6c: rsp+=1;stack[rsp]=0x23;
_0x6e: mem[0x4d]=stack[rsp];rsp-=1;
_0x70: rsp+=1;stack[rsp]=0x9e;
_0x72: mem[0x4e]=stack[rsp];rsp-=1;
_0x74: rsp+=1;stack[rsp]=0x27;
_0x76: mem[0x4f]=stack[rsp];rsp-=1;
_0x78: rsp+=1;stack[rsp]=0x60;
_0x7a: mem[0x50]=stack[rsp];rsp-=1;
_0x7c: rsp+=1;stack[rsp]=0x2f;
_0x7e: mem[0x51]=stack[rsp];rsp-=1;
_0x80: rsp+=1;stack[rsp]=0xa5;
_0x82: mem[0x52]=stack[rsp];rsp-=1;
_0x84: rsp+=1;stack[rsp]=0xcf;
_0x86: mem[0x53]=stack[rsp];rsp-=1;
_0x88: rsp+=1;stack[rsp]=0x1b;
_0x8a: mem[0x54]=stack[rsp];rsp-=1;
_0x8c: rsp+=1;stack[rsp]=0xbd;
_0x8e: mem[0x55]=stack[rsp];rsp-=1;
_0x90: rsp+=1;stack[rsp]=0x32;
_0x92: mem[0x56]=stack[rsp];rsp-=1;
_0x94: rsp+=1;stack[rsp]=0xdb;
_0x96: mem[0x57]=stack[rsp];rsp-=1;
_0x98: rsp+=1;stack[rsp]=0xff;
_0x9a: mem[0x58]=stack[rsp];rsp-=1;
_0x9c: rsp+=1;stack[rsp]=0x28;
_0x9e: mem[0x59]=stack[rsp];rsp-=1;
_0xa0: rsp+=1;stack[rsp]=0xa4;
_0xa2: mem[0x5a]=stack[rsp];rsp-=1;
_0xa4: rsp+=1;stack[rsp]=0x5d;
_0xa6: mem[0x5b]=stack[rsp];rsp-=1;
_0xa8: temp=getchar();rsp+=1;stack[rsp]=temp;
_0xa9: mem[0x64]=stack[rsp];rsp-=1;
_0xab: temp=getchar();rsp+=1;stack[rsp]=temp;
_0xac: mem[0x65]=stack[rsp];rsp-=1;
_0xae: temp=getchar();rsp+=1;stack[rsp]=temp;
_0xaf: mem[0x66]=stack[rsp];rsp-=1;
_0xb1: temp=getchar();rsp+=1;stack[rsp]=temp;
_0xb2: mem[0x67]=stack[rsp];rsp-=1;
_0xb4: temp=getchar();rsp+=1;stack[rsp]=temp;
_0xb5: mem[0x68]=stack[rsp];rsp-=1;
_0xb7: temp=getchar();rsp+=1;stack[rsp]=temp;
_0xb8: mem[0x69]=stack[rsp];rsp-=1;
_0xba: temp=getchar();rsp+=1;stack[rsp]=temp;
_0xbb: mem[0x6a]=stack[rsp];rsp-=1;
_0xbd: temp=getchar();rsp+=1;stack[rsp]=temp;
_0xbe: mem[0x6b]=stack[rsp];rsp-=1;
_0xc0: temp=getchar();rsp+=1;stack[rsp]=temp;
_0xc1: mem[0x6c]=stack[rsp];rsp-=1;
_0xc3: temp=getchar();rsp+=1;stack[rsp]=temp;
_0xc4: mem[0x6d]=stack[rsp];rsp-=1;
_0xc6: temp=getchar();rsp+=1;stack[rsp]=temp;
_0xc7: mem[0x6e]=stack[rsp];rsp-=1;
_0xc9: temp=getchar();rsp+=1;stack[rsp]=temp;
_0xca: mem[0x6f]=stack[rsp];rsp-=1;
_0xcc: temp=getchar();rsp+=1;stack[rsp]=temp;
_0xcd: mem[0x70]=stack[rsp];rsp-=1;
_0xcf: temp=getchar();rsp+=1;stack[rsp]=temp;
_0xd0: mem[0x71]=stack[rsp];rsp-=1;
_0xd2: temp=getchar();rsp+=1;stack[rsp]=temp;
_0xd3: mem[0x72]=stack[rsp];rsp-=1;
_0xd5: temp=getchar();rsp+=1;stack[rsp]=temp;
_0xd6: mem[0x73]=stack[rsp];rsp-=1;
_0xd8: temp=getchar();rsp+=1;stack[rsp]=temp;
_0xd9: mem[0x74]=stack[rsp];rsp-=1;
_0xdb: temp=getchar();rsp+=1;stack[rsp]=temp;
_0xdc: mem[0x75]=stack[rsp];rsp-=1;
_0xde: temp=getchar();rsp+=1;stack[rsp]=temp;
_0xdf: mem[0x76]=stack[rsp];rsp-=1;
_0xe1: temp=getchar();rsp+=1;stack[rsp]=temp;
_0xe2: mem[0x77]=stack[rsp];rsp-=1;
_0xe4: temp=getchar();rsp+=1;stack[rsp]=temp;
_0xe5: mem[0x78]=stack[rsp];rsp-=1;
_0xe7: temp=getchar();rsp+=1;stack[rsp]=temp;
_0xe8: mem[0x79]=stack[rsp];rsp-=1;
_0xea: temp=getchar();rsp+=1;stack[rsp]=temp;
_0xeb: mem[0x7a]=stack[rsp];rsp-=1;
_0xed: temp=getchar();rsp+=1;stack[rsp]=temp;
_0xee: mem[0x7b]=stack[rsp];rsp-=1;
_0xf0: temp=getchar();rsp+=1;stack[rsp]=temp;
_0xf1: mem[0x7c]=stack[rsp];rsp-=1;
_0xf3: temp=getchar();rsp+=1;stack[rsp]=temp;
_0xf4: mem[0x7d]=stack[rsp];rsp-=1;
_0xf6: temp=getchar();rsp+=1;stack[rsp]=temp;
_0xf7: mem[0x7e]=stack[rsp];rsp-=1;
_0xf9: temp=getchar();rsp+=1;stack[rsp]=temp;
_0xfa: mem[0x7f]=stack[rsp];rsp-=1;
_0xfc: temp=getchar();rsp+=1;stack[rsp]=temp;
_0xfd: mem[0x80]=stack[rsp];rsp-=1;
_0xff: temp=getchar();rsp+=1;stack[rsp]=temp;
_0x100: mem[0x81]=stack[rsp];rsp-=1;
_0x102: temp=getchar();rsp+=1;stack[rsp]=temp;
_0x103: mem[0x82]=stack[rsp];rsp-=1;
_0x105: temp=getchar();rsp+=1;stack[rsp]=temp;
_0x106: mem[0x83]=stack[rsp];rsp-=1;
_0x108: temp=getchar();rsp+=1;stack[rsp]=temp;
_0x109: mem[0x84]=stack[rsp];rsp-=1;
_0x10b: temp=getchar();rsp+=1;stack[rsp]=temp;
_0x10c: mem[0x85]=stack[rsp];rsp-=1;
_0x10e: temp=getchar();rsp+=1;stack[rsp]=temp;
_0x10f: mem[0x86]=stack[rsp];rsp-=1;
_0x111: temp=getchar();rsp+=1;stack[rsp]=temp;
_0x112: mem[0x87]=stack[rsp];rsp-=1;
_0x114: temp=getchar();rsp+=1;stack[rsp]=temp;
_0x115: mem[0x88]=stack[rsp];rsp-=1;
_0x117: temp=getchar();rsp+=1;stack[rsp]=temp;
_0x118: mem[0x89]=stack[rsp];rsp-=1;
_0x11a: temp=getchar();rsp+=1;stack[rsp]=temp;
_0x11b: mem[0x8a]=stack[rsp];rsp-=1;
_0x11d: temp=getchar();rsp+=1;stack[rsp]=temp;
_0x11e: mem[0x8b]=stack[rsp];rsp-=1;
_0x120: temp=getchar();rsp+=1;stack[rsp]=temp;
_0x121: mem[0x8c]=stack[rsp];rsp-=1;
_0x123: temp=getchar();rsp+=1;stack[rsp]=temp;
_0x124: mem[0x8d]=stack[rsp];rsp-=1;
_0x126: rsp+=1;stack[rsp]=0x0;
_0x128: cnt[0x0]=stack[rsp];rsp-=1;
_0x12a: rsp+=1;stack[rsp]=cnt[0x0];
_0x12c: rsp+=1;stack[rsp]=0x7;
_0x12e: rax=stack[rsp];rsp-=1;rbx=stack[rsp];rsp-=1;if (stack[rax]>stack[rbx]) goto _0x130;  else goto _0x184;
_0x130: rsp+=1;stack[rsp]=0x0;
_0x132: cnt[0x1]=stack[rsp];rsp-=1;
_0x134: rsp+=1;stack[rsp]=cnt[0x1];
_0x136: rsp+=1;stack[rsp]=0x6;
_0x138: rax=stack[rsp];rsp-=1;rbx=stack[rsp];rsp-=1;if (stack[rax]>stack[rbx]) goto _0x13a;  else goto _0x17a;
_0x13a: rsp+=1;stack[rsp]=cnt[0x0];
_0x13c: rsp+=1;stack[rsp]=0x6;
_0x13e: rax=stack[rsp];rsp-=1;rbx=stack[rsp];rsp-=1;rsp+=1;stack[rsp]=rbx*rax;
_0x13f: rsp+=1;stack[rsp]=cnt[0x1];
_0x141: rax=stack[rsp];rsp-=1;rbx=stack[rsp];rsp-=1;rsp+=1;stack[rsp]=rbx+rax;
_0x142: rsp+=1;stack[rsp]=0x64;
_0x144: rax=stack[rsp];rsp-=1;rbx=stack[rsp];rsp-=1;rsp+=1;stack[rsp]=rbx+rax;
_0x145: rax=stack[rsp];rsp-=1;rsp+=1;stack[rsp]=mem[rax];
_0x146: reg=stack[rsp];rsp-=1;rsp+=1;stack[rsp]=~(reg);
_0x147: rsp+=1;stack[rsp]=cnt[0x0];
_0x149: rsp+=1;stack[rsp]=cnt[0x1];
_0x14b: rsp+=1;stack[rsp]=0x2;
_0x14d: rax=stack[rsp];rsp-=1;rbx=stack[rsp];rsp-=1;rsp+=1;stack[rsp]=rbx+rax;
_0x14e: rax=stack[rsp];rsp-=1;rbx=stack[rsp];rsp-=1;rsp+=1;stack[rsp]=rbx*rax;
_0x14f: rax=stack[rsp];rsp-=1;rbx=stack[rsp];rsp-=1;rsp+=1;stack[rsp]=rbx&rax;
_0x150: rsp+=1;stack[rsp]=0x64;
_0x152: rsp+=1;stack[rsp]=cnt[0x0];
_0x154: rsp+=1;stack[rsp]=0x6;
_0x156: rax=stack[rsp];rsp-=1;rbx=stack[rsp];rsp-=1;rsp+=1;stack[rsp]=rbx*rax;
_0x157: rsp+=1;stack[rsp]=cnt[0x1];
_0x159: rax=stack[rsp];rsp-=1;rbx=stack[rsp];rsp-=1;rsp+=1;stack[rsp]=rbx+rax;
_0x15a: rax=stack[rsp];rsp-=1;rbx=stack[rsp];rsp-=1;rsp+=1;stack[rsp]=rbx+rax;
_0x15b: rax=stack[rsp];rsp-=1;rsp+=1;stack[rsp]=mem[rax];
_0x15c: rsp+=1;stack[rsp]=cnt[0x0];
_0x15e: rsp+=1;stack[rsp]=cnt[0x1];
_0x160: rsp+=1;stack[rsp]=0x2;
_0x162: rax=stack[rsp];rsp-=1;rbx=stack[rsp];rsp-=1;rsp+=1;stack[rsp]=rbx+rax;
_0x163: rax=stack[rsp];rsp-=1;rbx=stack[rsp];rsp-=1;rsp+=1;stack[rsp]=rbx*rax;
_0x164: reg=stack[rsp];rsp-=1;rsp+=1;stack[rsp]=~(reg);
_0x165: rax=stack[rsp];rsp-=1;rbx=stack[rsp];rsp-=1;rsp+=1;stack[rsp]=rbx&rax;
_0x166: rax=stack[rsp];rsp-=1;rbx=stack[rsp];rsp-=1;rsp+=1;stack[rsp]=rbx|rax;
_0x167: rsp+=1;stack[rsp]=cnt[0x1];
_0x169: rsp+=1;stack[rsp]=0x7;
_0x16b: rax=stack[rsp];rsp-=1;rbx=stack[rsp];rsp-=1;rsp+=1;stack[rsp]=rbx*rax;
_0x16c: rsp+=1;stack[rsp]=cnt[0x0];
_0x16e: rax=stack[rsp];rsp-=1;rbx=stack[rsp];rsp-=1;rsp+=1;stack[rsp]=rbx+rax;
_0x16f: rax=stack[rsp];rsp-=1;rbx=stack[rsp];rsp-=1;mem[rax]=rbx;
_0x170: rsp+=1;stack[rsp]=cnt[0x1];
_0x172: rsp+=1;stack[rsp]=0x1;
_0x174: rax=stack[rsp];rsp-=1;rbx=stack[rsp];rsp-=1;rsp+=1;stack[rsp]=rbx+rax;
_0x175: rsp+=1;stack[rsp]=0x1;
_0x177: rax=stack[rsp];rsp-=1;rbx=stack[rsp];rsp-=1;cnt[rax]=rbx;
_0x178: goto _0x134;
_0x17a: rsp+=1;stack[rsp]=cnt[0x0];
_0x17c: rsp+=1;stack[rsp]=0x1;
_0x17e: rax=stack[rsp];rsp-=1;rbx=stack[rsp];rsp-=1;rsp+=1;stack[rsp]=rbx+rax;
_0x17f: rsp+=1;stack[rsp]=0x0;
_0x181: rax=stack[rsp];rsp-=1;rbx=stack[rsp];rsp-=1;cnt[rax]=rbx;
_0x182: goto _0x12a;
_0x184: rsp+=1;stack[rsp]=0x1;
_0x186: cnt[0x0]=stack[rsp];rsp-=1;
_0x188: rsp+=1;stack[rsp]=cnt[0x0];
_0x18a: rsp+=1;stack[rsp]=0x2a;
_0x18c: rax=stack[rsp];rsp-=1;rbx=stack[rsp];rsp-=1;if (stack[rax]>stack[rbx]) goto _0x18e;  else goto _0x1c0;
_0x18e: rsp+=1;stack[rsp]=cnt[0x0];
_0x190: rsp+=1;stack[rsp]=0x2;
_0x192: rax=stack[rsp];rsp-=1;rbx=stack[rsp];rsp-=1;rsp+=1;stack[rsp]=rbx%rax;
_0x193: rsp+=1;stack[rsp]=0x0;
_0x195: rax=stack[rsp];rsp-=1;rbx=stack[rsp];rsp-=1;if (stack[rax]==stack[rbx]) goto _0x197;  else goto _0x1a4;
_0x197: rsp+=1;stack[rsp]=cnt[0x0];
_0x199: rax=stack[rsp];rsp-=1;rsp+=1;stack[rsp]=mem[rax];
_0x19a: rsp+=1;stack[rsp]=cnt[0x0];
_0x19c: rsp+=1;stack[rsp]=0x1;
_0x19e: rax=stack[rsp];rsp-=1;rbx=stack[rsp];rsp-=1;rsp+=1;stack[rsp]=rbx-rax;
_0x19f: rax=stack[rsp];rsp-=1;rsp+=1;stack[rsp]=mem[rax];
_0x1a0: rax=stack[rsp];rsp-=1;rbx=stack[rsp];rsp-=1;rsp+=1;stack[rsp]=rbx+rax;
_0x1a1: rsp+=1;stack[rsp]=cnt[0x0];
_0x1a3: rax=stack[rsp];rsp-=1;rbx=stack[rsp];rsp-=1;mem[rax]=rbx;
_0x1a4: rsp+=1;stack[rsp]=cnt[0x0];
_0x1a6: rsp+=1;stack[rsp]=0x2;
_0x1a8: rax=stack[rsp];rsp-=1;rbx=stack[rsp];rsp-=1;rsp+=1;stack[rsp]=rbx%rax;
_0x1a9: rsp+=1;stack[rsp]=0x1;
_0x1ab: rax=stack[rsp];rsp-=1;rbx=stack[rsp];rsp-=1;if (stack[rax]==stack[rbx]) goto _0x1ad;  else goto _0x1b6;
_0x1ad: rsp+=1;stack[rsp]=0x6b;
_0x1af: rsp+=1;stack[rsp]=cnt[0x0];
_0x1b1: rax=stack[rsp];rsp-=1;rsp+=1;stack[rsp]=mem[rax];
_0x1b2: rax=stack[rsp];rsp-=1;rbx=stack[rsp];rsp-=1;rsp+=1;stack[rsp]=rbx*rax;
_0x1b3: rsp+=1;stack[rsp]=cnt[0x0];
_0x1b5: rax=stack[rsp];rsp-=1;rbx=stack[rsp];rsp-=1;mem[rax]=rbx;
_0x1b6: rsp+=1;stack[rsp]=cnt[0x0];
_0x1b8: rsp+=1;stack[rsp]=0x1;
_0x1ba: rax=stack[rsp];rsp-=1;rbx=stack[rsp];rsp-=1;rsp+=1;stack[rsp]=rbx+rax;
_0x1bb: rsp+=1;stack[rsp]=0x0;
_0x1bd: rax=stack[rsp];rsp-=1;rbx=stack[rsp];rsp-=1;cnt[rax]=rbx;
_0x1be: goto _0x188;
_0x1c0: rsp+=1;stack[rsp]=0x0;
_0x1c2: cnt[0x0]=stack[rsp];rsp-=1;
_0x1c4: rsp+=1;stack[rsp]=cnt[0x0];
_0x1c6: rsp+=1;stack[rsp]=0x29;
_0x1c8: rax=stack[rsp];rsp-=1;rbx=stack[rsp];rsp-=1;if (stack[rax]
```

然后可得到 exe

### [](#六：重新分析，调试exe)六：重新分析，调试 exe

```
int __cdecl main(int argc, const char **argv, const char **envp)
{
  char mem[768]; // [rsp+20h] [rbp-60h] BYREF
  int cnt[256]; // [rsp+320h] [rbp+2A0h] BYREF
  int stack[66]; // [rsp+720h] [rbp+6A0h] BYREF
  int reg; // [rsp+82Ch] [rbp+7ACh]
  int b; // [rsp+830h] [rbp+7B0h]
  int a; // [rsp+834h] [rbp+7B4h]
  char temp; // [rsp+83Bh] [rbp+7BBh]
  int sp_; // [rsp+83Ch] [rbp+7BCh]
 
  _main(argc, argv, envp);
  memset(stack, 0, sizeof(stack));
  memset(cnt, 0, sizeof(cnt));
  memset(mem, 0, sizeof(mem));
  mem[50] = 0x66;
  mem[51] = 0x4E;
  mem[52] = -87;
  mem[53] = -3;
  mem[54] = 60;
  mem[55] = 85;
  mem[56] = -112;
  mem[57] = 36;
  mem[58] = 87;
  mem[59] = -10;
  mem[60] = 93;
  mem[61] = -79;
  mem[62] = 1;
  mem[63] = 32;
  mem[64] = -127;
  mem[65] = -3;
  mem[66] = 54;
  mem[67] = -87;
  mem[68] = 31;
  mem[69] = -95;
  mem[70] = 14;
  mem[71] = 13;
  mem[72] = 0x80;
  mem[73] = -113;
  mem[74] = -50;
  mem[75] = 119;
  mem[76] = -24;
  mem[77] = 35;
  mem[78] = -98;
  mem[79] = 39;
  mem[80] = 96;
  mem[81] = 47;
  mem[82] = -91;
  mem[83] = -49;
  mem[84] = 27;
  mem[85] = -67;
  mem[86] = 50;
  mem[87] = -37;
  mem[88] = -1;
  mem[89] = 40;
  mem[90] = -92;
  stack[0] = 93;
  mem[91] = 93;
  sp_ = -1;
  temp = getchar();
  stack[++sp_] = temp;
  mem[0x64] = stack[sp_--];
  temp = getchar();
  stack[++sp_] = temp;
  mem[101] = stack[sp_--];
  temp = getchar();
  stack[++sp_] = temp;
  mem[102] = stack[sp_--];
  temp = getchar();
  stack[++sp_] = temp;
  mem[103] = stack[sp_--];
  temp = getchar();
  stack[++sp_] = temp;
  mem[104] = stack[sp_--];
  temp = getchar();
  stack[++sp_] = temp;
  mem[105] = stack[sp_--];
  temp = getchar();
  stack[++sp_] = temp;
  mem[106] = stack[sp_--];
  temp = getchar();
  stack[++sp_] = temp;
  mem[107] = stack[sp_--];
  temp = getchar();
  stack[++sp_] = temp;
  mem[108] = stack[sp_--];
  temp = getchar();
  stack[++sp_] = temp;
  mem[109] = stack[sp_--];
  temp = getchar();
  stack[++sp_] = temp;
  mem[110] = stack[sp_--];
  temp = getchar();
  stack[++sp_] = temp;
  mem[111] = stack[sp_--];
  temp = getchar();
  stack[++sp_] = temp;
  mem[112] = stack[sp_--];
  temp = getchar();
  stack[++sp_] = temp;
  mem[113] = stack[sp_--];
  temp = getchar();
  stack[++sp_] = temp;
  mem[114] = stack[sp_--];
  temp = getchar();
  stack[++sp_] = temp;
  mem[115] = stack[sp_--];
  temp = getchar();
  stack[++sp_] = temp;
  mem[116] = stack[sp_--];
  temp = getchar();
  stack[++sp_] = temp;
  mem[117] = stack[sp_--];
  temp = getchar();
  stack[++sp_] = temp;
  mem[118] = stack[sp_--];
  temp = getchar();
  stack[++sp_] = temp;
  mem[119] = stack[sp_--];
  temp = getchar();
  stack[++sp_] = temp;
  mem[120] = stack[sp_--];
  temp = getchar();
  stack[++sp_] = temp;
  mem[121] = stack[sp_--];
  temp = getchar();
  stack[++sp_] = temp;
  mem[122] = stack[sp_--];
  temp = getchar();
  stack[++sp_] = temp;
  mem[123] = stack[sp_--];
  temp = getchar();
  stack[++sp_] = temp;
  mem[124] = stack[sp_--];
  temp = getchar();
  stack[++sp_] = temp;
  mem[125] = stack[sp_--];
  temp = getchar();
  stack[++sp_] = temp;
  mem[126] = stack[sp_--];
  temp = getchar();
  stack[++sp_] = temp;
  mem[127] = stack[sp_--];
  temp = getchar();
  stack[++sp_] = temp;
  mem[128] = stack[sp_--];
  temp = getchar();
  stack[++sp_] = temp;
  mem[129] = stack[sp_--];
  temp = getchar();
  stack[++sp_] = temp;
  mem[130] = stack[sp_--];
  temp = getchar();
  stack[++sp_] = temp;
  mem[131] = stack[sp_--];
  temp = getchar();
  stack[++sp_] = temp;
  mem[132] = stack[sp_--];
  temp = getchar();
  stack[++sp_] = temp;
  mem[133] = stack[sp_--];
  temp = getchar();
  stack[++sp_] = temp;
  mem[134] = stack[sp_--];
  temp = getchar();
  stack[++sp_] = temp;
  mem[135] = stack[sp_--];
  temp = getchar();
  stack[++sp_] = temp;
  mem[136] = stack[sp_--];
  temp = getchar();
  stack[++sp_] = temp;
  mem[137] = stack[sp_--];
  temp = getchar();
  stack[++sp_] = temp;
  mem[138] = stack[sp_--];
  temp = getchar();
  stack[++sp_] = temp;
  mem[139] = stack[sp_--];
  temp = getchar();
  stack[++sp_] = temp;
  mem[140] = stack[sp_--];
  temp = getchar();
  stack[++sp_] = temp;
  mem[141] = stack[sp_--];
  stack[++sp_] = 0;
  cnt[0] = stack[sp_--];
  while ( 1 )
  {
    stack[++sp_] = cnt[0];
    stack[++sp_] = 7;
    a = stack[sp_--];
    b = stack[sp_--];
    if ( stack[a] <= stack[b] )                 // 说明外面要循环7次 while(i<7)
      break;
    stack[++sp_] = 0;
    cnt[1] = stack[sp_--];
    while ( 1 )
    {
      stack[++sp_] = cnt[1];
      stack[++sp_] = 6;
      a = stack[sp_--];
      b = stack[sp_--];
      if ( stack[a] <= stack[b] )               // 说明内部要循环6次 while(j<6)
        break;
      stack[++sp_] = cnt[0];
      stack[++sp_] = 6;
      a = stack[sp_--];
      b = stack[sp_--];
      stack[++sp_] = a * b;                     // i*6
      stack[++sp_] = cnt[1];                    // j
      a = stack[sp_--];
      b = stack[sp_--];
      stack[++sp_] = a + b;                     // (i*6)+j
      stack[++sp_] = 100;
      a = stack[sp_--];
      b = stack[sp_--];
      stack[++sp_] = a + b;                     // (i*6+j)+100
      a = stack[sp_--];
      stack[++sp_] = mem[a];                    // input[(6*i+j)+100]
      reg = stack[sp_--];
      stack[++sp_] = ~reg;                      // ~input[(6*i+j)+100]
      stack[++sp_] = cnt[0];
      stack[++sp_] = cnt[1];
      stack[++sp_] = 2;                         // i j 2
      a = stack[sp_--];
      b = stack[sp_--];
      stack[++sp_] = a + b;                     // j+2
      a = stack[sp_--];
      b = stack[sp_--];
      stack[++sp_] = a * b;                     // (j+2)*i
      a = stack[sp_--];
      b = stack[sp_--];
      stack[++sp_] = a & b;                     // (~input[(6*i+j)+100] & ((j+2)*i))
      stack[++sp_] = 100;
      stack[++sp_] = cnt[0];
      stack[++sp_] = 6;
      a = stack[sp_--];                         // 6
      b = stack[sp_--];                         // i
      stack[++sp_] = a * b;
      stack[++sp_] = cnt[1];
      a = stack[sp_--];
      b = stack[sp_--];
      stack[++sp_] = a + b;                     // (6*i+j)
      a = stack[sp_--];
      b = stack[sp_--];
      stack[++sp_] = a + b;                     // (6*i+j)+100
      a = stack[sp_--];
      stack[++sp_] = mem[a];                    // input[(6*i+j)+100]
      stack[++sp_] = cnt[0];
      stack[++sp_] = cnt[1];
      stack[++sp_] = 2;
      a = stack[sp_--];
      b = stack[sp_--];
      stack[++sp_] = a + b;
      a = stack[sp_--];
      b = stack[sp_--];
      stack[++sp_] = a * b;
      reg = stack[sp_--];
      stack[++sp_] = ~reg;                      // ~((j+2)*i)
      a = stack[sp_--];
      b = stack[sp_--];
      stack[++sp_] = a & b;                     // (~input[(6*i+j)+100] & ((j+2)*i))
      a = stack[sp_--];
      b = stack[sp_--];
      stack[++sp_] = a | b;                     // (~input[(6*i+j)+100] & ((j+2)*i)) | (~input[(6*i+j)+100] & ((j+2)*i))
      stack[++sp_] = cnt[1];
      stack[++sp_] = 7;
      a = stack[sp_--];
      b = stack[sp_--];
      stack[++sp_] = a * b;
      stack[++sp_] = cnt[0];
      a = stack[sp_--];
      b = stack[sp_--];
      stack[++sp_] = a + b;                     // j*7+i
      a = stack[sp_--];
      b = stack[sp_--];
      mem[a] = b;                               // input[j*7+i] = (~input[(6*i+j)+100] & ((j+2)*i)) | (~input[(6*i+j)+100] & ((j+2)*i))
      stack[++sp_] = cnt[1];                    // 等价于input[j*7+i]和((j+2)*i)异或
      stack[++sp_] = 1;
      a = stack[sp_--];
      b = stack[sp_--];                         // j+=1
      stack[++sp_] = a + b;
      stack[++sp_] = 1;
      a = stack[sp_--];
      b = stack[sp_--];
      cnt[a] = b;
    }
    stack[++sp_] = cnt[0];                      // i += 1
    stack[++sp_] = 1;
    a = stack[sp_--];
    b = stack[sp_--];
    stack[++sp_] = a + b;
    stack[++sp_] = 0;
    a = stack[sp_--];
    b = stack[sp_--];
    cnt[a] = b;
  }
  stack[++sp_] = 1;
  cnt[0] = stack[sp_--];
  while ( 1 )
  {
    stack[++sp_] = cnt[0];
    stack[++sp_] = 42;
    a = stack[sp_--];
    b = stack[sp_--];
    if ( stack[a] <= stack[b] )                 // while(i < 42)
      break;
    stack[++sp_] = cnt[0];
    stack[++sp_] = 2;
    a = stack[sp_--];
    b = stack[sp_--];
    stack[++sp_] = b % a;
    stack[++sp_] = 0;
    a = stack[sp_--];
    b = stack[sp_--];
    if ( stack[a] == stack[b] )                 // if (i % 2 == 0)
    {
      stack[++sp_] = cnt[0];
      a = stack[sp_--];
      stack[++sp_] = mem[a];                    // input[i]
      stack[++sp_] = cnt[0];
      stack[++sp_] = 1;
      a = stack[sp_--];
      b = stack[sp_--];
      stack[++sp_] = b - a;
      a = stack[sp_--];
      stack[++sp_] = mem[a];                    // input[i-1]
      a = stack[sp_--];
      b = stack[sp_--];
      stack[++sp_] = a + b;                     // input[i] + input[i-1]
      stack[++sp_] = cnt[0];
      a = stack[sp_--];
      b = stack[sp_--];
      mem[a] = b;                               // input[i] = input[i] + input[i-1]
    }
    stack[++sp_] = cnt[0];
    stack[++sp_] = 2;
    a = stack[sp_--];
    b = stack[sp_--];
    stack[++sp_] = b % a;
    stack[++sp_] = 1;
    a = stack[sp_--];
    b = stack[sp_--];
    if ( stack[a] == stack[b] )                 // if (i % 2) == 1
    {
      stack[++sp_] = 107;
      stack[++sp_] = cnt[0];
      a = stack[sp_--];
      stack[++sp_] = mem[a];                    // input[i]
      a = stack[sp_--];
      b = stack[sp_--];
      stack[++sp_] = a * b;                     // input[i] * 107
      stack[++sp_] = cnt[0];
      a = stack[sp_--];
      b = stack[sp_--];
      mem[a] = b;                               // input[i] = input[i] * 107
    }
    stack[++sp_] = cnt[0];
    stack[++sp_] = 1;
    a = stack[sp_--];
    b = stack[sp_--];
    stack[++sp_] = a + b;                       // i+=1
    stack[++sp_] = 0;
    a = stack[sp_--];
    b = stack[sp_--];
    cnt[a] = b;
  }
  stack[++sp_] = 0;
  cnt[0] = stack[sp_--];
  while ( 1 )
  {
    stack[++sp_] = cnt[0];
    stack[++sp_] = 0x29;
    a = stack[sp_--];
    b = stack[sp_--];
    if ( stack[a] < stack[b] )
      break;
    stack[++sp_] = cnt[0];
    a = stack[sp_--];
    stack[++sp_] = mem[a];                      // mem[i]
    stack[++sp_] = 0x32;
    stack[++sp_] = cnt[0];
    a = stack[sp_--];
    b = stack[sp_--];
    stack[++sp_] = a + b;                       // [i+0x32]
    a = stack[sp_--];
    stack[++sp_] = mem[a];                      // mem[i+0x32]
    a = stack[sp_--];
    b = stack[sp_--];
    if ( stack[a] != stack[b] )                 // if (mem[i+0x32] != mem[i])
    {
      stack[++sp_] = 'n';
      putchar(stack[sp_--]);                    // printf("n");
      break;
    }
    stack[++sp_] = cnt[0];
    stack[++sp_] = 1;
    a = stack[sp_--];
    b = stack[sp_--];
    stack[++sp_] = a + b;
    stack[++sp_] = 0;
    a = stack[sp_--];
    b = stack[sp_--];
    cnt[a] = b;
  }
  stack[++sp_] = 'y';                           // printf("y");
  putchar(stack[sp_]);
  return 0;
}

```

**分析技巧，观察一下会发现都是 push push，然后 pop pop 之后进行运算这种，大脑里面想像一个栈，跟着看就行**

```
stack[++sp_] = cnt[0];
 stack[++sp_] = 7;
 a = stack[sp_--];
 b = stack[sp_--];
 if ( stack[a] <= stack[b] )                 // 说明外面要循环7次 while(i<7)
   break;

```

最后写出解题脚本

```
cmp_data = [0x66, 0x4E, 0xA9, 0xFD, 0x3C, 0x55, 0x90, 0x24, 0x57, 0xF6, 0x5D, 0xB1, 0x01, 0x20, 0x81, 0xFD, 0x36, 0xA9, 0x1F, 0xA1, 0x0E, 0x0D, 0x80, 0x8F, 0xCE, 0x77, 0xE8, 0x23, 0x9E, 0x27, 0x60, 0x2F, 0xA5, 0xCF, 0x1B, 0xBD, 0x32, 0xDB, 0xFF, 0x28, 0xA4, 0x5D]
data = [0]*42
for i in range(42):
    if (i % 2) == 0:
        data[i] = (cmp_data[i] - cmp_data[i-1])
    if (i % 2) == 1:
        for x in range(0xff):
            if (x * 107) & 0xff == cmp_data[i]:
                data[i] = x
flag = ''
for i in range(7):
    for j in range(6):
        flag += chr((data[j*7+i] ^ ((j + 2) * i)) & 0xff)
print(flag)
# flag{wh03v3r_d1g5_1n70_17_f1nd5_7h3_7ru7h}

```

[[看雪官方培训] Unicorn Trace 还原 Ollvm 算法！《安卓高级研修班》2021 年秋季班火热招生！！](https://bbs.pediy.com/thread-267018.htm)

上传的附件：

*   [VM.zip](javascript:void(0)) （1.59MB，7 次下载）