> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.52pojie.cn](https://www.52pojie.cn/thread-2017438-1-1.html)

> 看到分享了 8.5 安装包，之前的 kg 失效了，才发现替换成了 9.x 的注册模式。做了简单分析，整理如下：调试版本为 9.0（240905），新注册机制都一样，ida.dll 中导 ...

![](https://avatar.52pojie.cn/data/avatar/000/14/47/47_avatar_middle.jpg)Nisy _ 本帖最后由 Nisy 于 2025-3-22 12:11 编辑_  
看到分享了 8.5 安装包，之前的 kg 失效了，才发现替换成了 9.x 的注册模式。  
做了简单分析，整理如下：  
调试版本为 9.0（240905），新注册机制都一样，[IDA](https://www.52pojie.cn/thread-1874203-1-1.html).dll 中导出表搜索 license，“get_license_manager” 看到这个比较可疑，下断运行：  
  
// ida.dll                   | ModuleBase:7FF896E00000 SizeOfImage:48B000  

```
{
// IDA.DLL <get_license_manager>
00007FF896E366F0 | 48:895C24 08         | mov qword ptr ss:[rsp+8],rbx            | <get_license_manager>
00007FF896E366F5 | 48:897424 10         | mov qword ptr ss:[rsp+10],rsi           |
00007FF896E366FA | 48:897C24 18         | mov qword ptr ss:[rsp+18],rdi           |
...
00007FF896E36AE2 | 41:B9 10000000       | mov r9d,10                              |
00007FF896E36AE8 | 4C:8965 BF           | mov qword ptr ss:[rbp-41],r12           |
00007FF896E36AEC | 4C:8965 C7           | mov qword ptr ss:[rbp-39],r12           |
00007FF896E36AF0 | 4C:8965 CF           | mov qword ptr ss:[rbp-31],r12           |
00007FF896E36AF4 | 48:8B0D 4DE54100     | mov rcx,qword ptr ds:[7FF897255048]     |
00007FF896E36AFB | 48:8B01              | mov rax,qword ptr ds:[rcx]              | "D:\\Program Files\\IDA Professional 9.0\\ida.exe"
00007FF896E36AFE | 48:8D55 BF           | lea rdx,qword ptr ss:[rbp-41]           |
00007FF896E36B02 | 48:895424 20         | mov qword ptr ss:[rsp+20],rdx           |
00007FF896E36B07 | 4C:8D45 1F           | lea r8,qword ptr ss:[rbp+1F]            |
00007FF896E36B0B | 49:8BD5              | mov rdx,r13                             |
00007FF896E36B0E | FF50 18              | call qword ptr ds:[rax+18]              | 这里开始计算 CALL 00007FF8971333F0 【见下文函数1】
00007FF896E36B11 | 85C0                 | test eax,eax                            |
00007FF896E36B13 | 74 1A                | je ida.7FF896E36B2F                     |
...
}
```

```
{ // 【函数1】带函数读取 keyfile 并进行验证
// 00007FF896E36B0E         call qword ptr ds:[rax+18]  
00007FF8971333F0 | 40:55                | push rbp                                | 这里开始验证
00007FF8971333F2 | 53                   | push rbx                                |
00007FF8971333F3 | 56                   | push rsi                                |
...
00007FF897133B19 | 4C:8BC6              | mov r8,rsi                              | "D:\\Program Files\\IDA Professional 9.0\\idapro_96-2923-BE84-E1.hexlic", 
00007FF897133B1C | 48:8D5424 38         | lea rdx,qword ptr ss:[rsp+38]           |
00007FF897133B21 | 48:8BCF              | mov rcx,rdi                             |
00007FF897133B24 | E8 37BEFFFF          | call <ida.sub_7FF89712F960>             | 判断文件是否存在
{
        00007FF89712F9DB | 48:8B0E              | mov rcx,qword ptr ds:[rsi]              | "D:\\Program Files\\IDA Professional 9.0\\idapro_96-2923-BE84-E1.hexlic"
        00007FF89712F9DE | E8 8D55FCFF          | call <ida.qbasename>                    | 取文件名
        00007FF89712F9E3 | BA 2E000000          | mov edx,2E                              | 2E:'.'
        00007FF89712F9E8 | 48:8BC8              | mov rcx,rax                             | rcx:".hexlic", rax:".hexlic"
        00007FF89712F9EB | E8 4C2C0100          | call <JMP.&strrchr>                     |
        00007FF89712F9F0 | 48:85C0              | test rax,rax                            | rax:".hexlic"
        00007FF89712F9F3 | 0F84 F2010000        | je ida.7FF89712FBEB                     |
        00007FF89712F9F9 | 48:8D15 08CE0900     | lea rdx,qword ptr ds:[7FF8971CC808]     | rdx:".hexlic"
        00007FF89712FA00 | 48:8BC8              | mov rcx,rax                             | rcx:".hexlic"
        00007FF89712FA03 | FF15 0F2F0600        | call qword ptr ds:[<_stricmp>]          | 比较后缀名
}
00007FF897133B2B | 75 1E                | jne ida.7FF897133B4B                    |
00007FF897133B2D | 48:8B4C24 38         | mov rcx,qword ptr ss:[rsp+38]           |
00007FF897133B32 | E8 C9C8FBFF          | call <ida.qfree>                        |
00007FF897133B37 | 90                   | nop                                     |
00007FF897133B38 | 48:8BCB              | mov rcx,rbx                             |
00007FF897133B3B | E8 B06EFCFF          | call <ida.qmutex_unlock>                |
00007FF897133B40 | 90                   | nop                                     |
00007FF897133B41 | 44:8B6424 30         | mov r12d,dword ptr ss:[rsp+30]          |
00007FF897133B46 | E9 C7020000          | jmp ida.7FF897133E12                    |
00007FF897133B4B | 48:8D4424 38         | lea rax,qword ptr ss:[rsp+38]           |
00007FF897133B50 | 48:3BF0              | cmp rsi,rax                             |
00007FF897133B53 | 48:8B4424 40         | mov rax,qword ptr ss:[rsp+40]           |
00007FF897133B58 | 74 2D                | je ida.7FF897133B87                     |
00007FF897133B5A | 4C:8D40 FF           | lea r8,qword ptr ds:[rax-1]             | r8:&"{\n  "payload": {\n    "name": "Nisy/PYG",
00007FF897133B5E | 48:85C0              | test rax,rax                            |
00007FF897133B61 | B9 00000000          | mov ecx,0                               |
00007FF897133B66 | 4C:0F44C1            | cmove r8,rcx                            | r8:&"{\n  "payload": {\n    "name": "Nisy/PYG",
00007FF897133B6A | 4D:85C0              | test r8,r8                              | r8:&"{\n  "payload": {\n    "name": "Nisy/PYG\
00007FF897133B6D | 74 14                | je ida.7FF897133B83                     |
00007FF897133B6F | 48:8B5424 38         | mov rdx,qword ptr ss:[rsp+38]           |
00007FF897133B74 | 48:8BCE              | mov rcx,rsi                             |
00007FF897133B77 | E8 84EECCFF          | call ida.7FF896E02A00                   |
00007FF897133B7C | 48:8B4424 40         | mov rax,qword ptr ss:[rsp+40]           |
00007FF897133B81 | EB 04                | jmp ida.7FF897133B87                    |
00007FF897133B83 | 48:894E 08           | mov qword ptr ds:[rsi+8],rcx            |
00007FF897133B87 | 48:8D35 5DF90500     | lea rsi,qword ptr ds:[7FF8971934EB]     |
00007FF897133B8E | 4C:8BCE              | mov r9,rsi                              |
00007FF897133B91 | 48:85C0              | test rax,rax                            |
00007FF897133B94 | 4C:0F454C24 38       | cmovne r9,qword ptr ss:[rsp+38]         |
00007FF897133B9A | 4C:8D87 58010000     | lea r8,qword ptr ds:[rdi+158]           | r8:&"{\n  "payload": {\n    "name": "Nisy/PYG"
00007FF897133BA1 | 48:8D97 88000000     | lea rdx,qword ptr ds:[rdi+88]           |
00007FF897133BA8 | 4C:896C24 28         | mov qword ptr ss:[rsp+28],r13           |
00007FF897133BAD | 44:8B6424 34         | mov r12d,dword ptr ss:[rsp+34]          |
00007FF897133BB2 | 44:896424 20         | mov dword ptr ss:[rsp+20],r12d          |
00007FF897133BB7 | 48:8D4F 70           | lea rcx,qword ptr ds:[rdi+70]           |
00007FF897133BBB | E8 E094FFFF          | call <ida.sub_7FF89712D0A0>             | 读取 KEYFILE 并进行验证【见下文函数2】
00007FF897133BC0 | 85C0                 | test eax,eax                            |
00007FF897133BC2 | 0F85 2C020000        | jne ida.7FF897133DF4                    |
00007FF897133BC8 | 41:F6C4 10           | test r12b,10                            |
00007FF897133BCC | 0F84 22020000        | je ida.7FF897133DF4                     |
...
}
```

```
{ // 【函数2】打开 KEYFILE 后并进行验证
// 00007FF897133BBB  call <ida.sub_7FF89712D0A0>

00007FF89712D0A0 | 4C:8BDC              | mov r11,rsp                             |
00007FF89712D0A3 | 49:895B 08           | mov qword ptr ds:[r11+8],rbx            |
00007FF89712D0A7 | 49:896B 10           | mov qword ptr ds:[r11+10],rbp           |
...
00007FF89712D0E5 | E8 26040000          | call <ida.sub_7FF89712D510>             | 读取 file
{
        00007FF89712D510 | 48:895C24 18         | mov qword ptr ss:[rsp+18],rbx           | [qword ptr ss:[rsp+18]]:_malloc_base+36
        00007FF89712D515 | 57                   | push rdi                                |
        00007FF89712D516 | 48:83EC 20           | sub rsp,20                              |
        00007FF89712D51A | 49:8BD8              | mov rbx,r8                              |
        00007FF89712D51D | 4D:8BC8              | mov r9,r8                               |
        00007FF89712D520 | 41:B8 00001000       | mov r8d,100000                          |
        00007FF89712D526 | 48:8BF9              | mov rdi,rcx                             |
        00007FF89712D529 | E8 22D9FCFF          | call <ida.sub_7FF8970FAE50>             | 读取 keyfile 【见下文函数3】
        00007FF89712D52E | 84C0                 | test al,al                              |
        00007FF89712D530 | 74 1C                | je ida.7FF89712D54E                     |
        00007FF89712D532 | 48:8BD3              | mov rdx,rbx                             |
        00007FF89712D535 | 48:8BCF              | mov rcx,rdi                             |
        00007FF89712D538 | E8 23020000          | call <ida.sub_7FF89712D760>             | 校验数据
        00007FF89712D53D | 84C0                 | test al,al                              |
        00007FF89712D53F | 74 0D                | je ida.7FF89712D54E                     |
        00007FF89712D541 | B0 01                | mov al,1                                |
        00007FF89712D543 | 48:8B5C24 40         | mov rbx,qword ptr ss:[rsp+40]           |
        00007FF89712D548 | 48:83C4 20           | add rsp,20                              |
        00007FF89712D54C | 5F                   | pop rdi                                 |
        00007FF89712D54D | C3                   | ret                                     |
}
00007FF89712D0EA | 84C0                 | test al,al                              |
00007FF89712D0EC | 75 07                | jne ida.7FF89712D0F5                    |
00007FF89712D0EE | BB 02000000          | mov ebx,2                               |
00007FF89712D0F3 | EB 76                | jmp ida.7FF89712D16B                    |
00007FF89712D0F5 | 48:897424 20         | mov qword ptr ss:[rsp+20],rsi           |
00007FF89712D0FA | 44:8B8C24 80000000   | mov r9d,dword ptr ss:[rsp+80]           |
00007FF89712D102 | 4C:8D4424 30         | lea r8,qword ptr ss:[rsp+30]            | R8 是 data
00007FF89712D107 | 48:8BD3              | mov rdx,rbx                             | rdx:"D:\\Program Files\\IDA Professional 9.0\\idapro_96-2923-BE84-E1.hexlic"
00007FF89712D10A | 49:8BCE              | mov rcx,r14                             |
00007FF89712D10D | E8 EED9FFFF          | call <ida.sub_7FF89712AB00>             |  ### 核心验证函数 开始计算 keyfile Data
{
        00007FF89712AB3F | 49:8D4B D8           | lea rcx,qword ptr ds:[r11-28]           |
        00007FF89712AB43 | E8 F8180000          | call <ida.sub_7FF89712C440>             | 这里是验证算法，返回 TURE 为验证成功 【见下文函数4】
        00007FF89712AB48 | 84C0                 | test al,al                              | 返回 AL = 1，注册验证成功
        00007FF89712AB4A | 74 1C                | je ida.7FF89712AB68                     |
        00007FF89712AB4C | 48:895C24 20         | mov qword ptr ss:[rsp+20],rbx           |
        00007FF89712AB51 | 45:33C9              | xor r9d,r9d                             |
        00007FF89712AB54 | 4C:8D4424 30         | lea r8,qword ptr ss:[rsp+30]            |
        00007FF89712AB59 | 48:8BD7              | mov rdx,rdi                             | rdx:"D:\\Program Files\\IDA Professional 9.0\\idapro_96-2923-BE84-E1.hexlic"
        00007FF89712AB5C | 48:8BCE              | mov rcx,rsi                             |
        00007FF89712AB5F | E8 DC1D0000          | call <ida.sub_7FF89712C940>             | 解析并使用 keyfile JSON 字段
        00007FF89712AB64 | 8BE8                 | mov ebp,eax                             | 文件解析的结果
        00007FF89712AB66 | EB 05                | jmp ida.7FF89712AB6D                    |
}
00007FF89712D112 | 8BD8                 | mov ebx,eax                             |
00007FF89712D114 | 48:85FF              | test rdi,rdi                            |
00007FF89712D117 | 74 34                | je ida.7FF89712D14D                     |
...
}
```

```
{ // 【函数3】调用 API 打开 keyfile 并读取内容
// 00007FF89712D529  call <ida.sub_7FF8970FAE50>        读取 keyfile

00007FF8970FAE50 | 48:895C24 08         | mov qword ptr ss:[rsp+8],rbx            | 读取 license
00007FF8970FAE55 | 48:896C24 10         | mov qword ptr ss:[rsp+10],rbp           |
00007FF8970FAE5A | 48:897424 18         | mov qword ptr ss:[rsp+18],rsi           | [qword ptr ss:[rsp+18]]:&"D:\\Program Files\\IDA Professional 9.0\\idapro_96-2923-BE84-E1.hexlic"
00007FF8970FAE5F | 57                   | push rdi                                |
00007FF8970FAE60 | 41:56                | push r14                                |
00007FF8970FAE62 | 41:57                | push r15                                | r15:&"D:\\Program Files\\IDA Professional 9.0\\idapro_96-2923-BE84-E1.hexlic"
00007FF8970FAE64 | 48:83EC 20           | sub rsp,20                              |
00007FF8970FAE68 | 48:8BF2              | mov rsi,rdx                             | rdx:"D:\\Program Files\\IDA Professional 9.0\\idapro_96-2923-BE84-E1.hexlic"
00007FF8970FAE6B | 4D:8BF0              | mov r14,r8                              |
00007FF8970FAE6E | 4C:8BF9              | mov r15,rcx                             | r15:&"D:\\Program Files\\IDA Professional 9.0\\idapro_96-2923-BE84-E1.hexlic"
00007FF8970FAE71 | 48:8D15 54DA0C00     | lea rdx,qword ptr ds:[7FF8971C88CC]     | rdx:"D:\\Program Files\\IDA Professional 9.0\\idapro_96-2923-BE84-E1.hexlic",
00007FF8970FAE78 | 48:8BCE              | mov rcx,rsi                             |
00007FF8970FAE7B | 41:B8 40000000       | mov r8d,40                              | 40:'@'
00007FF8970FAE81 | 49:8BF9              | mov rdi,r9                              |
00007FF8970FAE84 | E8 C7FEFFFF          | call <ida.sub_7FF8970FAD50>             | 这里打开 license
{
        00007FF8970FAD7A | 48:8D4C24 20         | lea rcx,qword ptr ss:[rsp+20]           | [ss:[rsp+20]]:&"D:\\Program Files\\IDA Professional 9.0\\idapro_96-2923-BE84-E1.hexlic"
        00007FF8970FAD7F | E8 9C93FFFF          | call <ida.utf8_utf16>                   |
        00007FF8970FAD84 | 84C0                 | test al,al                              |
        00007FF8970FAD86 | 74 4E                | je ida.7FF8970FADD6                     |
        00007FF8970FAD88 | 48:8BD7              | mov rdx,rdi                             | rdx:"D:\\Program Files\\IDA Professional 9.0\\idapro_96-2923-BE84-E1.hexlic"
        00007FF8970FAD8B | 48:8D4C24 38         | lea rcx,qword ptr ss:[rsp+38]           |
        00007FF8970FAD90 | E8 3BC4FFFF          | call <ida.sub_7FF8970F71D0>             |
        00007FF8970FAD95 | 90                   | nop                                     |
        00007FF8970FAD96 | 48:8B4424 40         | mov rax,qword ptr ss:[rsp+40]           |
        00007FF8970FAD9B | 48:83F8 01           | cmp rax,1                               |
        00007FF8970FAD9F | 76 2A                | jbe ida.7FF8970FADCB                    |
        00007FF8970FADA1 | 48:8D0D B84D0A00     | lea rcx,qword ptr ds:[7FF89719FB60]     |
        00007FF8970FADA8 | 48:8BD1              | mov rdx,rcx                             | rdx:"D:\\Program Files\\IDA Professional 9.0\\idapro_96-2923-BE84-E1.hexlic"
        00007FF8970FADAB | 48:395C24 28         | cmp qword ptr ss:[rsp+28],rbx           |
        00007FF8970FADB0 | 48:0F455424 20       | cmovne rdx,qword ptr ss:[rsp+20]        | [qword ptr ss:[rsp+20]]:&"D:\\Program Files\\IDA Professional 9.0\\idapro_96-2923-BE84-E1.hexlic"
        00007FF8970FADB6 | 48:85C0              | test rax,rax                            |
        00007FF8970FADB9 | 48:0F454C24 38       | cmovne rcx,qword ptr ss:[rsp+38]        |
        00007FF8970FADBF | 44:8BC6              | mov r8d,esi                             |
        00007FF8970FADC2 | FF15 107A0900        | call qword ptr ds:[<_wfsopen>]          |  调用 CreateFileW 打开 license
}
00007FF8970FAE89 | 48:8BE8              | mov rbp,rax                             |
00007FF8970FAE8C | 48:85C0              | test rax,rax                            |
00007FF8970FAE8F | 0F84 BA000000        | je ida.7FF8970FAF4F                     |
00007FF8970FAE95 | 48:8BC8              | mov rcx,rax                             |
00007FF8970FAE98 | 40:32F6              | xor sil,sil                             |
00007FF8970FAE9B | E8 D0120000          | call <ida.qfsize>                       |
00007FF8970FAEA0 | 48:8BD8              | mov rbx,rax                             |
00007FF8970FAEA3 | 48:85C0              | test rax,rax                            |
00007FF8970FAEA6 | 7E 68                | jle ida.7FF8970FAF10                    |
00007FF8970FAEA8 | 49:3BC6              | cmp rax,r14                             |
00007FF8970FAEAB | 7F 63                | jg ida.7FF8970FAF10                     |
00007FF8970FAEAD | 48:8BD0              | mov rdx,rax                             | rdx:"D:\\Program Files\\IDA Professional 9.0\\idapro_96-2923-BE84-E1.hexlic"
00007FF8970FAEB0 | 49:8BCF              | mov rcx,r15                             | r15:&"D:\\Program Files\\IDA Professional 9.0\\idapro_96-2923-BE84-E1.hexlic"
00007FF8970FAEB3 | E8 482FD1FF          | call <ida.sub_7FF896E0DE00>             |
00007FF8970FAEB8 | 49:8B17              | mov rdx,qword ptr ds:[r15]              | rdx:"D:\\Program Files\\IDA Professional 9.0\\idapro_96-2923-BE84-E1.hexlic", 
00007FF8970FAEBE | 48:8BCD              | mov rcx,rbp                             |
00007FF8970FAEC1 | E8 7A040000          | call <ida.qfread>                       | 读取到 RDX 中
00007FF8970FAEC6 | 48:3BC3              | cmp rax,rbx                             |
00007FF8970FAEC9 | 75 05                | jne ida.7FF8970FAED0                    |
}
```

```
{ // 【函数4】这里是进入关键算法
// 00007FF89712AB43  call <ida.sub_7FF89712C440>             | 这里是验证算法，返回 TURE 为验证成功
...
00007FF89712C4E7 | 48:8D4C24 20         | lea rcx,qword ptr ss:[rsp+20]           |
00007FF89712C4EC | E8 BF77F8FF          | call <ida.parse_json_string>            | 解析完 json
00007FF89712C4F1 | 85C0                 | test eax,eax                            |
00007FF89712C4F3 | 75 5C                | jne ida.7FF89712C551                    |
00007FF89712C4F5 | 837C24 20 03         | cmp dword ptr ss:[rsp+20],3             |
00007FF89712C4FA | 75 55                | jne ida.7FF89712C551                    |
00007FF89712C4FC | 4C:8BC5              | mov r8,rbp                              |
00007FF89712C4FF | 41:8BD6              | mov edx,r14d                            |
00007FF89712C502 | 48:8B4C24 28         | mov rcx,qword ptr ss:[rsp+28]           |
00007FF89712C507 | E8 94000000          | call <ida.sub_7FF89712C5A0>             | ### 注册算法验证 ###【见下文函数5】
00007FF89712C50C | 85C0                 | test eax,eax                            | 这里返回 EAX = 0 则代表成功
00007FF89712C50E | 75 41                | jne ida.7FF89712C551                    | 这里不跳
00007FF89712C510 | 48:85DB              | test rbx,rbx                            |
00007FF89712C513 | 74 38                | je ida.7FF89712C54D                     |
00007FF89712C515 | 837C24 20 03         | cmp dword ptr ss:[rsp+20],3             |
00007FF89712C51A | 75 6C                | jne ida.7FF89712C588                    |
00007FF89712C51C | 4C:8B0B              | mov r9,qword ptr ds:[rbx]               |
00007FF89712C51F | 4C:8B43 08           | mov r8,qword ptr ds:[rbx+8]             | r8:&"{\n  "payload": {\n    "name": "Nisy/PYG"
00007FF89712C523 | 48:8B53 10           | mov rdx,qword ptr ds:[rbx+10]           |
00007FF89712C527 | 48:8B4C24 28         | mov rcx,qword ptr ss:[rsp+28]           |
00007FF89712C52C | 48:8B01              | mov rax,qword ptr ds:[rcx]              |
00007FF89712C52F | 48:8903              | mov qword ptr ds:[rbx],rax              |
00007FF89712C532 | 48:8B41 08           | mov rax,qword ptr ds:[rcx+8]            |
00007FF89712C536 | 48:8943 08           | mov qword ptr ds:[rbx+8],rax            |
00007FF89712C53A | 48:8B41 10           | mov rax,qword ptr ds:[rcx+10]           |
00007FF89712C53E | 48:8943 10           | mov qword ptr ds:[rbx+10],rax           |
00007FF89712C542 | 4C:8909              | mov qword ptr ds:[rcx],r9               |
00007FF89712C545 | 4C:8941 08           | mov qword ptr ds:[rcx+8],r8             | r8:&"{\n  "payload": {\n    "name": "Nisy/PYG"
00007FF89712C549 | 48:8951 10           | mov qword ptr ds:[rcx+10],rdx           |
00007FF89712C54D | B3 01                | mov bl,1                                | 验证成功赋值为 1
00007FF89712C54F | EB 02                | jmp ida.7FF89712C553                    |
00007FF89712C551 | 32DB                 | xor bl,bl                               |
...                                 |
00007FF89712C569 | 0FB6C3               | movzx eax,bl                            |
00007FF89712C56C | 48:8B5C24 70         | mov rbx,qword ptr ss:[rsp+70]           | [qword ptr ss:[rsp+70]]:writebytes+10D2B
00007FF89712C571 | 48:8B6C24 78         | mov rbp,qword ptr ss:[rsp+78]           | [qword ptr ss:[rsp+78]]:&"D:\\Program Files\\IDA Professional 9.0\\idapro_96-2923-BE84-E1.hexlic"
00007FF89712C576 | 48:8BB424 80000000   | mov rsi,qword ptr ss:[rsp+80]           |
00007FF89712C57E | 48:83C4 50           | add rsp,50                              |
00007FF89712C582 | 41:5F                | pop r15                                 | r15:&"D:\\Program Files\\IDA Professional 9.0\\idapro_96-2923-BE84-E1.hexlic"
00007FF89712C584 | 41:5E                | pop r14                                 |
00007FF89712C586 | 5F                   | pop rdi                                 |
00007FF89712C587 | C3                   | ret                                     |
}
```

```
{【函数5】读取 JSON 中的 payload 对象，序列化后计算 SHA256 的哈希值，并用公钥(N,e)解密 signature 对象的 HEX 字符串值，将解密的数据[0x20~0x40)和 SHA256 数值做比较，若相同则验证成功
// 00007FF89712C507     call <ida.sub_7FF89712C5A0>  

00007FF89712C5A0 | 48:895C24 10         | mov qword ptr ss:[rsp+10],rbx           | 进入验证函数
00007FF89712C5A5 | 55                   | push rbp                                |
00007FF89712C5A6 | 56                   | push rsi                                |
...
00007FF89712C5F0 | 4C:396B 08           | cmp qword ptr ds:[rbx+8],r13            |
00007FF89712C5F4 | 48:8D0D F06E0600     | lea rcx,qword ptr ds:[7FF8971934EB]     |
00007FF89712C5FB | 74 03                | je ida.7FF89712C600                     |
00007FF89712C5FD | 48:8B0B              | mov rcx,qword ptr ds:[rbx]              |
00007FF89712C600 | 48:8D15 41FA0900     | lea rdx,qword ptr ds:[7FF8971CC048]     | 找到 JSON 中 signature 字段
00007FF89712C607 | E8 84600100          | call <JMP.&strcmp>                      |
00007FF89712C60C | 85C0                 | test eax,eax                            |
00007FF89712C60E | 74 4D                | je ida.7FF89712C65D                     | 找到之后，跳到新篇章
00007FF89712C610 | 48:FFC7              | inc rdi                                 |
00007FF89712C613 | 48:83C3 28           | add rbx,28                              | rbx 为当前 JSON 的字段信息: payload\header\signature
00007FF89712C617 | 48:3BFE              | cmp rdi,rsi                             | 枚举 JSON 中的字段（3个）
00007FF89712C61A | 72 D4                | jb ida.7FF89712C5F0                     |
...
00007FF89712C65D | 48:8D3CBF            | lea rdi,qword ptr ds:[rdi+rdi*4]        |
00007FF89712C661 | 48:8D7F 03           | lea rdi,qword ptr ds:[rdi+3]            |
00007FF89712C665 | 49:8D3CFF            | lea rdi,qword ptr ds:[r15+rdi*8]        | [ds:[r15+rdi*8]]:"D:\\Program Files\\IDA Professional 9.0\\idapro_96-2923-BE84-E1.hexlic"
00007FF89712C669 | 833F 02              | cmp dword ptr ds:[rdi],2                |
00007FF89712C66C | 75 AE                | jne ida.7FF89712C61C                    |
00007FF89712C66E | 48:85FF              | test rdi,rdi                            |
00007FF89712C671 | 74 A9                | je ida.7FF89712C61C                     |
00007FF89712C673 | 49:8BD6              | mov rdx,r14                             |
00007FF89712C676 | 48:8D4D 80           | lea rcx,qword ptr ss:[rbp-80]           |
00007FF89712C67A | E8 210B0000          | call <ida.sub_7FF89712D1A0>             | 读取 payload 并计算序列化后的 sha256
{
        00007FF89712D1A0 | 48:895C24 18         | mov qword ptr ss:[rsp+18],rbx           | [qword ptr ss:[rsp+18]]:_malloc_base+36
        00007FF89712D1A5 | 55                   | push rbp                                |
        00007FF89712D1A6 | 56                   | push rsi                                |
        ...
        00007FF89712D357 | EB 04                | jmp ida.7FF89712D35D                    |
        00007FF89712D359 | 4C:8959 08           | mov qword ptr ds:[rcx+8],r11            |
        00007FF89712D35D | 49:8D56 18           | lea rdx,qword ptr ds:[r14+18]           |
        00007FF89712D361 | 48:8BCB              | mov rcx,rbx                             |
        00007FF89712D364 | E8 A763F8FF          | call <ida.jvalue_t_copy>                |
        00007FF89712D369 | 4C:8B6C24 30         | mov r13,qword ptr ss:[rsp+30]           | [qword ptr ss:[rsp+30]]:"{\n  "payload": {\n    "name": "Nisy/PYG"      "issued_on": "2025-02-28 03:21:00",\n        "license_type": "computer",\n       
        00007FF89712D36E | 48:8B7424 28         | mov rsi,qword ptr ss:[rsp+28]           |
        00007FF89712D373 | 4C:8B6424 20         | mov r12,qword ptr ss:[rsp+20]           |
        00007FF89712D378 | 48:8D3D 71EF0900     | lea rdi,qword ptr ds:[7FF8971CC2F0]     | ds:[00007FF8971CC2F0]:"header"
        00007FF89712D37F | 49:83C6 28           | add r14,28                              |
        00007FF89712D383 | 4C:3B7424 60         | cmp r14,qword ptr ss:[rsp+60]           |
        00007FF89712D388 | 0F85 92FEFFFF        | jne ida.7FF89712D220                    |
        00007FF89712D38E | 48:8B7C24 40         | mov rdi,qword ptr ss:[rsp+40]           |
        00007FF89712D393 | 48:8D5424 20         | lea rdx,qword ptr ss:[rsp+20]           | 序列化 "payload"
        00007FF89712D398 | 48:8D4C24 68         | lea rcx,qword ptr ss:[rsp+68]           | 返回的 BUFFER 存储序列化的数据
        00007FF89712D39D | E8 4E020000          | call <ida.sub_7FF89712D5F0>             | 拼装 payload 并序列化 JSON
        {
                00007FF89712D5F0 | 48:8BC4              | mov rax,rsp                             |
                ...
                00007FF89712D629 | 44:8D46 02           | lea r8d,qword ptr ds:[rsi+2]            |
                00007FF89712D62D | 48:8D50 E0           | lea rdx,qword ptr ds:[rax-20]           | 传入的 rcx 指向的buffer 接受 json
                00007FF89712D631 | E8 4A67F8FF          | call <ida.serialize_json>               | 序列化 payload
                00007FF89712D636 | 0FB6F8               | movzx edi,al                            |
                00007FF89712D639 | 837C24 28 03         | cmp dword ptr ss:[rsp+28],3             |
                00007FF89712D63E | 75 41                | jne ida.7FF89712D681                    |
                00007FF89712D640 | 48:897424 30         | mov qword ptr ss:[rsp+30],rsi           | [qword ptr ss:[rsp+30]]:"{\n  "payload": {\n    "name": "Nisy/PYG"
                00007FF89712D645 | 897424 28            | mov dword ptr ss:[rsp+28],esi           |
                00007FF89712D649 | 48:8D4C24 28         | lea rcx,qword ptr ss:[rsp+28]           |
                00007FF89712D64E | E8 4D5FF8FF          | call <ida.jvalue_t_clear>               |
        }
        ...
        00007FF89712D416 | 48:8D4D 90           | lea rcx,qword ptr ss:[rbp-70]           |
        00007FF89712D41A | E8 717CFEFF          | call <ida.sha256_init>                  |  调用 SHA256 算法，计算 JSON 序列化后字符串的 HASH
        00007FF89712D41F | 44:8B4424 50         | mov r8d,dword ptr ss:[rsp+50]           | SIZE
        00007FF89712D424 | 48:8B5424 48         | mov rdx,qword ptr ss:[rsp+48]           | BUFFER
        00007FF89712D429 | 48:8D4D 90           | lea rcx,qword ptr ss:[rbp-70]           |
        00007FF89712D42D | E8 9E7CFEFF          | call <ida.sha256_update>                |
        00007FF89712D432 | 48:8D55 90           | lea rdx,qword ptr ss:[rbp-70]           |
        00007FF89712D436 | 48:8D4D 00           | lea rcx,qword ptr ss:[rbp]              |
        00007FF89712D43A | E8 517AFEFF          | call <ida.sha256_final>                 | 计算 JSON 序列化 的HASH
        00007FF89712D43F | 4C:8937              | mov qword ptr ds:[rdi],r14              |
        00007FF89712D442 | 4C:8977 08           | mov qword ptr ds:[rdi+8],r14            |
        00007FF89712D446 | 4C:8977 10           | mov qword ptr ds:[rdi+10],r14           |
        00007FF89712D44A | BA 20000000          | mov edx,20                              | 20:' '
        00007FF89712D44F | 48:8BCF              | mov rcx,rdi                             |
        00007FF89712D452 | E8 A909CEFF          | call <ida.sub_7FF896E0DE00>             |
        00007FF89712D457 | 48:8B07              | mov rax,qword ptr ds:[rdi]              |
        00007FF89712D45A | 0F1045 00            | movups xmm0,xmmword ptr ss:[rbp]        |
        00007FF89712D45E | 0F1100               | movups xmmword ptr ds:[rax],xmm0        |
        00007FF89712D461 | 0F104D 10            | movups xmm1,xmmword ptr ss:[rbp+10]     | 取 sha256
        00007FF89712D465 | 0F1148 10            | movups xmmword ptr ds:[rax+10],xmm1     |
        00007FF89712D469 | C74424 38 05000000   | mov dword ptr ss:[rsp+38],5             |
        00007FF89712D471 | 48:8B4C24 48         | mov rcx,qword ptr ss:[rsp+48]           | [qword ptr ss:[rsp+48]]:_realloc_base+55
        ...
}
00007FF89712C67F | 90                   | nop                                     | RAX = sha356 vector
00007FF89712C680 | 49:8BDD              | mov rbx,r13                             |
00007FF89712C683 | 48:895C24 50         | mov qword ptr ss:[rsp+50],rbx           |
...
00007FF89712C715 | 4C:8D4424 30         | lea r8,qword ptr ss:[rsp+30]            |
00007FF89712C71A | 48:8D15 CB6D0600     | lea rdx,qword ptr ds:[7FF8971934EC]     | rdx:"%02X", ds:[00007FF8971934EC]:"%02X"
00007FF89712C721 | 48:8BCF              | mov rcx,rdi                             | rcx:"07A541E2623A2E09146E5DE9519FAB0280206AB40E168D34F7294E286A
00007FF89712C724 | E8 D744FCFF          | call <ida.qsscanf>                      |
00007FF89712C729 | 83F8 01              | cmp eax,1                               |
00007FF89712C72C | 75 19                | jne ida.7FF89712C747                    |
00007FF89712C72E | 8B4424 30            | mov eax,dword ptr ss:[rsp+30]           |
00007FF89712C732 | 3D FF000000          | cmp eax,FF                              |
00007FF89712C737 | 77 0E                | ja ida.7FF89712C747                     |
00007FF89712C739 | 8803                 | mov byte ptr ds:[rbx],al                | 这里赋值
00007FF89712C73B | 48:83C7 02           | add rdi,2                               | rdi:"07A541E2623A2E09146E5DE9519FAB0280206AB40E168D34F7
00007FF89712C73F | 48:FFC3              | inc rbx                                 |
00007FF89712C742 | 48:3BDE              | cmp rbx,rsi                             |
00007FF89712C745 | 75 C9                | jne ida.7FF89712C710                    | 把 signature 变成 HEX
...
00007FF89712C784 | 49:81F8 80000000     | cmp r8,80                               | hex size
00007FF89712C78B | 0F85 77010000        | jne ida.7FF89712C908                    |
00007FF89712C791 | 48:8D05 B8FC0900     | lea rax,qword ptr ds:[7FF8971CC450]     | RSA N
00007FF89712C798 | 48:894424 28         | mov qword ptr ss:[rsp+28],rax           |
00007FF89712C79D | 48:8D05 4CFD0900     | lea rax,qword ptr ds:[7FF8971CC4F0]     | RSA e， 0x13
00007FF89712C7A4 | 48:894424 20         | mov qword ptr ss:[rsp+20],rax           |
00007FF89712C7A9 | 4D:8BC8              | mov r9,r8                               | R8 是 signature HEX
00007FF89712C7AC | 4C:8BC3              | mov r8,rbx                              | R9 = 80
00007FF89712C7AF | 41:8D51 20           | lea edx,qword ptr ds:[r9+20]            | 80 + 20 ==> A0
00007FF89712C7B3 | 48:8D4D A0           | lea rcx,qword ptr ss:[rbp-60]           | 
00007FF89712C7B7 | E8 F4B70000          | call <ida.sub_7FF897137FB0>             | 关键算法 CALL，直接 RSA 公钥解密运算【见下文函数6】
00007FF89712C7BC | 85C0                 | test eax,eax                            |  函数返回值
00007FF89712C7BE | 0F88 CE000000        | js ida.7FF89712C892                     |
00007FF89712C7C4 | B9 40000000          | mov ecx,40                              | 40:'@'
00007FF89712C7C9 | 48:98                | cdqe                                    |
00007FF89712C7CB | 48:3BC8              | cmp rcx,rax                             |
00007FF89712C7CE | 73 13                | jae ida.7FF89712C7E3                    |
00007FF89712C7D0 | 807C0D A0 00         | cmp byte ptr ss:[rbp+rcx-60],0          | 80
00007FF89712C7D5 | 0F85 B2000000        | jne ida.7FF89712C88D                    |
00007FF89712C7DB | 48:FFC1              | inc rcx                                 |
00007FF89712C7DE | 48:3BC8              | cmp rcx,rax                             |
00007FF89712C7E1 | 72 ED                | jb ida.7FF89712C7D0                     |
00007FF89712C7E3 | 4C:896C24 38         | mov qword ptr ss:[rsp+38],r13           |
00007FF89712C7E8 | 4C:896C24 40         | mov qword ptr ss:[rsp+40],r13           |
00007FF89712C7ED | 4C:896C24 48         | mov qword ptr ss:[rsp+48],r13           | [qword ptr ss:[rsp+48]]:_realloc_base+55
00007FF89712C7F2 | 33D2                 | xor edx,edx                             |
00007FF89712C7F4 | 44:8D4A 01           | lea r9d,qword ptr ds:[rdx+1]            |
00007FF89712C7F8 | 44:8D42 20           | lea r8d,qword ptr ds:[rdx+20]           |
00007FF89712C7FC | 48:8D4C24 38         | lea rcx,qword ptr ss:[rsp+38]           |
00007FF89712C801 | E8 7A3DFCFF          | call <ida.qvector_reserve>              |
00007FF89712C806 | 48:894424 38         | mov qword ptr ss:[rsp+38],rax           |
00007FF89712C80B | 41:B8 20000000       | mov r8d,20                              | 20:' '
00007FF89712C811 | 48:8B4C24 40         | mov rcx,qword ptr ss:[rsp+40]           |
00007FF89712C816 | 4C:2BC1              | sub r8,rcx                              | r8:&"{\n  "payload": {\n    "name": "Nisy/PYG", 
00007FF89712C819 | 48:03C8              | add rcx,rax                             |
00007FF89712C81C | 33D2                 | xor edx,edx                             |
00007FF89712C81E | E8 0D5E0100          | call <JMP.&memset>                      |
00007FF89712C823 | 48:C74424 40 2000000 | mov qword ptr ss:[rsp+40],20            | 20:' '
00007FF89712C82C | 48:8B4424 38         | mov rax,qword ptr ss:[rsp+38]           |
00007FF89712C831 | 0F2845 C0            | movaps xmm0,xmmword ptr ss:[rbp-40]     |
00007FF89712C835 | 0F1100               | movups xmmword ptr ds:[rax],xmm0        |
00007FF89712C838 | 0F284D D0            | movaps xmm1,xmmword ptr ss:[rbp-30]     |
00007FF89712C83C | 0F1148 10            | movups xmmword ptr ds:[rax+10],xmm1     |
00007FF89712C840 | 48:8B7424 38         | mov rsi,qword ptr ss:[rsp+38]           |
00007FF89712C845 | 48:897424 68         | mov qword ptr ss:[rsp+68],rsi           |
00007FF89712C84A | 4C:896C24 38         | mov qword ptr ss:[rsp+38],r13           |
00007FF89712C84F | 48:8B5C24 40         | mov rbx,qword ptr ss:[rsp+40]           |
00007FF89712C854 | 48:895C24 70         | mov qword ptr ss:[rsp+70],rbx           | [qword ptr ss:[rsp+70]]:writebytes+10D2B
00007FF89712C859 | 4C:896C24 40         | mov qword ptr ss:[rsp+40],r13           |
00007FF89712C85E | 48:8B4424 48         | mov rax,qword ptr ss:[rsp+48]           | [qword ptr ss:[rsp+48]]:_realloc_base+55
00007FF89712C863 | 48:894424 78         | mov qword ptr ss:[rsp+78],rax           | [qword ptr ss:[rsp+78]]:&"D:\\Program Files\\IDA Professional 9.0\\idapro_96-2923-BE84-E1.hexlic"
00007FF89712C868 | 4C:896C24 48         | mov qword ptr ss:[rsp+48],r13           | [qword ptr ss:[rsp+48]]:_realloc_base+55
00007FF89712C86D | 33C9                 | xor ecx,ecx                             |
00007FF89712C86F | E8 8C3BFCFF          | call <ida.qfree>                        |
00007FF89712C874 | 90                   | nop                                     |
00007FF89712C875 | 4C:8B45 88           | mov r8,qword ptr ss:[rbp-78]            |
00007FF89712C879 | 4C:3BC3              | cmp r8,rbx                              | size = 0x20 
00007FF89712C87C | 74 53                | je ida.7FF89712C8D1                     |  进入比较
00007FF89712C87E | 32C0                 | xor al,al                               | 比较失败，赋值为0
00007FF89712C880 | BF 0A000000          | mov edi,A                               | 0A:'\n'
00007FF89712C885 | 84C0                 | test al,al                              |
00007FF89712C887 | 41:0F45FD            | cmovne edi,r13d                         |
00007FF89712C88B | EB 1C                | jmp ida.7FF89712C8A9                    |
00007FF89712C88D | B8 FEFFFFFF          | mov eax,FFFFFFFE                        |
00007FF89712C892 | 44:8BC0              | mov r8d,eax                             |
00007FF89712C895 | 48:8D15 F4FC0900     | lea rdx,qword ptr ds:[7FF8971CC590]     | ds:[00007FF8971CC590]:"Signature decryption failed with code: %d"
00007FF89712C89C | 49:8BCC              | mov rcx,r12                             |
00007FF89712C89F | E8 DC75CDFF          | call ida.7FF896E03E80                   |
00007FF89712C8A4 | BF 0A000000          | mov edi,A                               | 0A:'\n'
00007FF89712C8A9 | 48:8BCE              | mov rcx,rsi                             |
00007FF89712C8AC | E8 4F3BFCFF          | call <ida.qfree>                        |
00007FF89712C8B1 | 90                   | nop                                     |
00007FF89712C8B2 | 48:8B5C24 50         | mov rbx,qword ptr ss:[rsp+50]           |
00007FF89712C8B7 | 48:8BCB              | mov rcx,rbx                             |
00007FF89712C8BA | E8 413BFCFF          | call <ida.qfree>                        |
00007FF89712C8BF | 90                   | nop                                     |
00007FF89712C8C0 | 48:8B4D 80           | mov rcx,qword ptr ss:[rbp-80]           |
00007FF89712C8C4 | E8 373BFCFF          | call <ida.qfree>                        |
00007FF89712C8C9 | 90                   | nop                                     |
00007FF89712C8CA | 8BC7                 | mov eax,edi                             |
00007FF89712C8CC | E9 65FDFFFF          | jmp ida.7FF89712C636                    |
00007FF89712C8D1 | 4D:85C0              | test r8,r8                              |
00007FF89712C8D4 | 74 23                | je ida.7FF89712C8F9                     |
00007FF89712C8D6 | 48:8B4D 80           | mov rcx,qword ptr ss:[rbp-80]           | 读取 SHA256 的数值
00007FF89712C8DA | 4C:8BC9              | mov r9,rcx                              |
00007FF89712C8DD | 48:8BD6              | mov rdx,rsi                             |
00007FF89712C8E0 | 48:2BD1              | sub rdx,rcx                             | RCX 中存放的计算出来的结果
00007FF89712C8E3 | 0FB6040A             | movzx eax,byte ptr ds:[rdx+rcx]         | 这里就可以比较了
00007FF89712C8E7 | 3801                 | cmp byte ptr ds:[rcx],al                | 比较 SHA1 和 RSA 值
00007FF89712C8E9 | 75 93                | jne ida.7FF89712C87E                    |
00007FF89712C8EB | 48:FFC1              | inc rcx                                 |
00007FF89712C8EE | 48:8BC1              | mov rax,rcx                             |
00007FF89712C8F1 | 49:2BC1              | sub rax,r9                              |
00007FF89712C8F4 | 49:3BC0              | cmp rax,r8                              | R8 = 0x20
00007FF89712C8F7 | 72 EA                | jb ida.7FF89712C8E3                     |
00007FF89712C8F9 | B0 01                | mov al,1                                | 比较成功，赋值为 
00007FF89712C8FB | BF 0A000000          | mov edi,A                               | 0A:'\n'
00007FF89712C900 | 84C0                 | test al,al                              |
00007FF89712C902 | 41:0F45FD            | cmovne edi,r13d                         |
00007FF89712C906 | EB A1                | jmp ida.7FF89712C8A9                    |
00007FF89712C908 | 803D 650C1300 00     | cmp byte ptr ds:[<under_debugger>],0    |
00007FF89712C90F | 74 01                | je ida.7FF89712C912                     |
00007FF89712C911 | CC                   | int3                                    |
00007FF89712C912 | B9 490C0000          | mov ecx,C49                             |
00007FF89712C917 | E8 7432FCFF          | call <ida.interr>                       |
00007FF89712C91C | 90                   | nop                                     |
00007FF89712C91D | 803D 500C1300 00     | cmp byte ptr ds:[<under_debugger>],0    |
00007FF89712C924 | 74 01                | je ida.7FF89712C927                     |
00007FF89712C926 | CC                   | int3                                    |
00007FF89712C927 | B9 57060000          | mov ecx,657                             |
00007FF89712C92C | E8 5F32FCFF          | call <ida.interr>                       |
}
```

> ```
> { // 【函数6】将 signature 对象的 HEX 字符串，用公钥（N，e）解密 
> // 00007FF89712C7B7   call <ida.sub_7FF897137FB0>
> 
> 00007FF897137FB0 | 40:53                | push rbx                                | 这里是 RSA
> 00007FF897137FB2 | 55                   | push rbp                                |
> 00007FF897137FB3 | 56                   | push rsi                                |
> ...
> 00007FF89713804C | 48:8D8C24 D0000000   | lea rcx,qword ptr ss:[rsp+D0]           | Signature
> 00007FF897138054 | E8 EFA50000          | call <JMP.&memcpy>                      | signature HEX
> 00007FF897138059 | 4C:8BCE              | mov r9,rsi                              | R9 = N
> 00007FF89713805C | 4D:8BC6              | mov r8,r14                              | R8 = E
> 00007FF89713805F | 48:8D9424 D0000000   | lea rdx,qword ptr ss:[rsp+D0]           | message HEX
> 00007FF897138067 | 48:8D4C24 30         | lea rcx,qword ptr ss:[rsp+30]           |
> 00007FF89713806C | E8 2F010000          | call ida.7FF8971381A0                   | GMP::mpz_powm
> 
> // RSA 公钥解密后的原始信息
> 000000CBEF3FE960  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................  
> 000000CBEF3FE970  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................  
> 000000CBEF3FE980  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................  
> 000000CBEF3FE990  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 9B  ................  
> 000000CBEF3FE9A0  BA C2 1E 52 84 B2 23 7A EF AE 98 9D 16 AF D1 E9 
> 000000CBEF3FE9B0  78 76 E5 19 D1 D3 F7 47 AE E7 8A 07 9C BB 4E 56  
> 000000CBEF3FE9C0  C4 74 2A 90 BA 97 18 0C 21 54 44 5E 7B 34 B4 A5  
> 000000CBEF3FE9D0  CF 9D 65 F7 F2 A6 11 96 B4 1A B5 3D 8C 20 A8 00  
> 
> 反转后格式为：
> {
>  BYTE szHex[0x20];
>  BYTE szSha256[0x20]; // keyfile JSON 格式化之后的 HASH
>  BYTE szTemp[0x3F];
> }
> 
> 000000CBEF3FEC30  A8 20 8C 3D B5 1A B4 96 11 A6 F2 F7 65 9D CF A5 
> 000000CBEF3FEC40  B4 34 7B 5E 44 54 21 0C 18 97 BA 90 2A 74 C4 56 
> 000000CBEF3FEC50  4E BB 9C 07 8A E7 AE 47 F7 D3 D1 19 E5 76 78 E9 
> 000000CBEF3FEC60  D1 AF 16 9D 98 AE EF 7A 23 B2 84 52 1E C2 BA 9B 
> 000000CBEF3FEC70  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 
> 000000CBEF3FEC80  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 
> 000000CBEF3FEC90  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 
> 000000CBEF3FECA0  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 
> 
> }
> ```

  
8.5 & 9.X 的新算法和 8.4 之前的类似，中规中矩的文件数据 HASH 后 RSA 加密，连 N 和 e 都懒得换：  
00007FF89076C450  ED FD 42 5C F9 78 54 6E 89 11 22 58 84 43 6C 57  
00007FF89076C460  14 05 25 65 0B CF 6E BF E8 0E DB C5 FB 1D E6 8F  
00007FF89076C470  4C 66 C2 9C B2 2E B6 68 78 8A FC B0 AB BB 71 80  
00007FF89076C480  44 58 4B 81 0F 89 70 CD DF 22 73 85 F7 5D 5D DD  
00007FF89076C490  D9 1D 4F 18 93 7A 08 AA 83 B2 8C 49 D1 2D C9 2E  
00007FF89076C4A0  75 05 BB 38 80 9E 91 BD 0F BD 2F 2E 6A B1 D2 E3  
00007FF89076C4B0  3C 0C 55 D5 BD DD 47 8E E8 BF 84 5F CE F3 C8 2B  
00007FF89076C4C0  9D 29 29 EC B7 1F 4D 1B 3D B9 6E 3A 8E 7A AF 93  
注意，内存中的大数，在使用中要先倒序。HEX 数值去空格和倒序可以用这款小工具：  
![](https://attach.52pojie.cn/forum/202503/22/013554etym18nsu2ymmdym.png)

**3.png** _(175.13 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc2NDc4M3wyMzFhZGJmMXwxNzQyNjE2ODIyfDB8MjAxNzQzOA%3D%3D&nothumb=yes)

2025-3-22 01:35 上传

  
用 RSA 替换工具计算一组 new N 和 new D：  
![](https://attach.52pojie.cn/forum/202503/22/013604twd7bd65bo5uq5ut.png)

**4.png** _(306.33 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc2NDc4NHw3MGVlNDk5OHwxNzQyNjE2ODIyfDB8MjAxNzQzOA%3D%3D&nothumb=yes)

2025-3-22 01:36 上传

  
替换位置 125 ：42 --> 01  
new N: 93AF7A8E3A6EB93D1B4D1FB7EC29299D2BC8F3CE5F84BFE88E47DDBDD5550C3CE3D2B16A2E2FBD0FBD919E8038BB05752EC92DD1498CB283AA087A93184F1DD9DD5D5DF7857322DFCD70890F814B58448071BBABB0FC8A7868B62EB29CC2664C8FE61DFBC5DB0EE8BF6ECF0B65250514576C4384582211896E5478F95C01FDED  
new P: 2F  
new Q: 3246A18D038D2EB834DE570D973423F424B15871D442FA85A2D1A8150CB5768C8EEB212997E4AD4C2ABF3082CC81074E0508925223EE8906B1B6A18EABADFF44B84B5BE7538EAF46BDC4447236E8A0C59E1BD2F3A3BD75F285B0670BC86DC0C85C8FB30EE6C2671E3646672A01C058E0CC0F1189A62C4754E9A4ED448C420A3  
new D: 81540140A2661EEAB06B19321E36F76EF955ABEEEA6FA97CD04DD71B3337D7B04DF46B336A1807E3EDF5EA42FD37B2CA61289356A9E748E71D5AD8D91B4EBE1E3F202AB9CB269DD29AAD151F109D244D2FFA1997C89177D33994EE1FFE861D61B835042EC0C18CA03E2442A3A2D227FEE47E5737FA9DB0C129DC768CF644D34F  
校验结果：成功！  
替换位置 125 ：42 --> DC  
new N: 93AF7A8E3A6EB93D1B4D1FB7EC29299D2BC8F3CE5F84BFE88E47DDBDD5550C3CE3D2B16A2E2FBD0FBD919E8038BB05752EC92DD1498CB283AA087A93184F1DD9DD5D5DF7857322DFCD70890F814B58448071BBABB0FC8A7868B62EB29CC2664C8FE61DFBC5DB0EE8BF6ECF0B65250514576C4384582211896E5478F95CDCFDED  
new D: 4DBAAC4ADB62B2560E5E7C7BBFA9E001E126655F24CC9AE62FEFEDF81F7021636A6EE41CEFE33B15C216BF3602E92B4B26190AA40BC3507B3111EFABBBF3BEDE7481FB8FBF7FF768512DC16679F1C2AACA56CE90423412FC013776E4BE4B5E433E433833AB80C47A7FB39564502E6E767EDAAA45A7A6242D627D4D24ED81C903  
校验结果：成功！  
计算成功总数：2  
总耗时：0 秒 125 毫秒  
线程数: 8  
素数集：2~65521 区间内的素数  
素数个数：6542  
选第一组，注意 Patch 点有两处：  
**// ED FD 42 5C F9 ==> ED FD 01 5C F9**  
![](https://attach.52pojie.cn/forum/202503/22/013645h11lotprtueeqyut.png)

**2.png** _(205.83 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc2NDc4NXxhOWFiMDY2NXwxNzQyNjE2ODIyfDB8MjAxNzQzOA%3D%3D&nothumb=yes)

2025-3-22 01:36 上传

  
然后就用 cursor 参考 9.x 的正版 key 写一下注册机格式：  
![](https://attach.52pojie.cn/forum/202503/22/013702zj6cn84i64nze6hj.png)

**6.png** _(233.66 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc2NDc4NnwwZGFmZmI0Y3wxNzQyNjE2ODIyfDB8MjAxNzQzOA%3D%3D&nothumb=yes)

2025-3-22 01:37 上传

  
生成之后提示 version 不对，猜一下，9.x 是 "version": 1，这里可能是 "version": 0，一试果然。  
再次提示缺少 ”data“，搜索对比下字符串，原来是替换了这里：  

```
Invalid license type: "%s"        
Invalid product type: "%s"        
Invalid edition:   "%s"   
Invalid IDA Home edition  
Invalid  issued timestamp  : "%s"  
Invalid   add-on entry      Features should   only be strings   
header  
Missing   "header" entry    
version 
Missing   "version" entry   
Missing, or unsupported version:  %lld   
payload   
-------------------------------------------------
// 9.x
Missing "payload  " entry 
// 8.5
Missing "data  " entry         // new
-------------------------------------------------
Missing   "name" entry      
email   
Missing   "email" entry     
licenses          
Missing "licenses" entry          
Badly formatted   license entry     
Failed to load file:    The file "%s" doesn't appear to be a valid license file
```

总结差异点：  
![](https://attach.52pojie.cn/forum/202503/22/014611xckn0anxzbgogke0.png)

**7.png** _(73.73 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc2NDc4N3xmZTIyODBlOHwxNzQyNjE2ODIyfDB8MjAxNzQzOA%3D%3D&nothumb=yes)

2025-3-22 01:46 上传

  
分析完成后，剩下的就是用 cursor 生成 keygen 代码。IDA 的验证流程中规中矩，将 keyfile 中 JSON 的 payload 对象序列化的字符串计算 SHA256 的哈希值，并用公钥（N，e）解密 signature 对象的 HEX 字符串值，将解密的数据 [0x20~0x40) 和 SHA256 数值做比较，若相同则验证成功。测试效果如下：  
![](https://attach.52pojie.cn/forum/202503/22/013857vazr7v87vjv7bzrf.jpg)

**8.jpg** _(52.1 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc2NDc4OXwxZWRjYWMyZXwxNzQyNjE2ODIyfDB8MjAxNzQzOA%3D%3D&nothumb=yes)

2025-3-22 01:38 上传![](https://avatar.52pojie.cn/images/noavatar_middle.gif)guodebin 学习 ing，顶一下![](https://avatar.52pojie.cn/data/avatar/000/15/59/43_avatar_middle.jpg)浙江 - 杺庝 校长的速度真快 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) YG2025 大牛厉害![](https://static.52pojie.cn/static/image/smiley/default/42.gif)，要是多点文字介绍就更好了 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) dengbin 今天又是元气满满的一天！![](https://avatar.52pojie.cn/images/noavatar_middle.gif)laochu 每天感觉看完都元气满满![](https://avatar.52pojie.cn/images/noavatar_middle.gif)秋名山 膜拜校长，如此高效 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) ziyouzizai 加点文字简介更好了 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) xiaohan231 太厉害了，学习一下 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) yiyepz 这个厉害了。。。