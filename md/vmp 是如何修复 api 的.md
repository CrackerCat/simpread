> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-260458.htm)

> vmp 是如何修复 api 的

1.vmp 在执行完形如以下指令时 完成了对 api 的解密以及调用

```
lea reg,[imm0]
mov reg,[reg+imm1]
lea reg,[reg+imm2]
xchg [rsp],reg
ret

```

2. 先说 imm1 因为 reg+imm1 位于 vmp0 区段之中  
eg:  
vmp0 的区段 base 以及 size 分别为

```
base = 000000014009D000
size = 00000000000B1000

```

```
lea rax, ds:[0x0000000140006A16]
// rax == 0x0000000140006A16
mov rax, qword ptr ds:[rax+0xEA94A]
// rax+0xEA94A = 00000001400F1360
// [rax+0xEA94A] = [00000001400F1360] = 00007FFDE207E8F0
lea rax, ds:[rax-0x3262A0B0]
// rax = 00007FFDAFA54840

```

00007FFDAFA54840 为 jmp qword ptr ds:[<&GetModuleFileNameW>]

 

此时 kernel32.dll 的 base 为

```
base=00007FFDAFA30000

```

设想以下情况:  
假如电脑关机后重启 重新打开 而此时 kernel32.dll 的 base 变成了

```
base=00007FFC36780000

```

那此时 [00000001400F1360] 应该填上多少比较合适呢

```
[00000001400F1360] = 00007FFC36780000 - 00007FFDAFA30000 + 00007FFDE207E8F0 = 00007FFC68DCE8F0

```

至此就修复了该 api 的调用

 

所以我们是不是就可以这样了  
弄点代码 跑在原本 oep 之前 根据记录到的 dllname,dllbase,dllsize 以及 api 的 name  
然后在对 vmp0 进行暴搜  
进行修复就可以了

 

回到最开始 我们发现还有个 imm2 而这个立即数会导致  
[reg+imm1] 里面存放的数可能不属于 [dllbase, dllbase + dllsize]

 

3. 看看 vmp 是如何完成映射的

 

写入

```
// mov qword ptr ds:[rax],rdx
// rdx == 0x0000000020A44E60
// rax == 0x000007FEEAC2C999

```

先看看 映射空间 0x000007FEEAC2C999 怎么计算得到的

```
add qword ptr ss:[rbp+8],rax
// [rbp+8] == 0x000007FEEAC2C999 =  0x7FD6AB50000 + 0x1800DC999

```

rax 来自于 mov rax,qword ptr ss:[rbp]

```
mov rax,qword ptr ds:[rsi]
// rax == 0x2033B8E1C16B3FB
add rax,rbx
// rax == 0x2033B9013000000
ror rax,14
// rax == 0x2033B90130
bswap rax
// rax == 0x3001B93320000000
rol rax,23
// rax == 0x1800DC999
mov qword ptr ss:[rbp],rax
// [rbp] == 0x1800DC999

```

我们不难发现 这个偏移来自于 vmp 的 bytecode 里面  
再看看 数据 0x0000000020A44E60 怎么计算得到的

```
mov rax,qword ptr ss:[rbp]
// rax == 0xFFFFFFFFA90235F0
// [rbp+8] == 0x77A21870
add qword ptr ss:[rbp+8],rax
// [rbp+8] == 0x20A44E60

```

先看 0xFFFFFFFFA90235F0

```
mov eax,dword ptr ds:[rsi]
// eax == 0x45BC37E0
xor eax,ebx
// eax == 0x1A574EB1
inc eax
// eax == 0x1A574EB2
sub eax,94C212A
// rax == 0xA90235F0
cdqe
// rax == 0xFFFFFFFFA90235F0
mov qword ptr ss:[rbp],rax
// [rbp] == 0xFFFFFFFFA90235F0

```

偏移也来自于 vmp 的 bytecode 里面

 

再看关键的 0x77A21870  
首先 kernel32.dll 的 base 为 0x77A10000

```
// rdx == 77AB003C
    // 为Export directory for KERNEL32.dll

```

```
mov ebx,dword ptr ds:[rdx+20]
// ebx == 0xA1610
    // 获取到 AddressOfNames
add rbx,rax 
// rbx == 0x77AB1610
    // Export Names Table for KERNEL32.dll

```

```
mov edi,dword ptr ds:[rdx+24]
// edi == 0xA2BBC 
    // 获取到 AddressOfNameOrdinals
add rdi,rax
// rdi == 0x77AB2BBC == 0xA2BBC + 0x77A10000
    // 获取到Export Ordinals Table for KERNEL32.dll
---
movzx ecx,word ptr ds:[rdi+rcx*2]
// ecx == 0x8F
mov edi,dword ptr ds:[rdx+1C]
// edi == 0xA0064
add rdi,rax
// rdi == 0x77AB0064
    // 获取到 Export Address Table for KERNEL32.dll
mov edi,dword ptr ds:[rdi+rcx*4]
// edi == 0x11870
add rax,rdi
// rax == 0x77A21870 == 0x77A10000 + 0x11870
// 0000000077A21870 ; HANDLE __stdcall CreateFileWImplementation

```

接下来 我们看看获得序号 0x8F 后做了哪些事情

```
add ecx,1
// ecx == 8F
mov dword ptr ss:[rbp-8],ecx
mov ecx,dword ptr ss:[rbp-8]
add ecx,dword ptr ss:[rbp-4]
// ecx == 11E
shr ecx,1
// ecx == 8F
mov edi,dword ptr ds:[rbx+rcx*4]
// edi == 0xA4258
add rdi,rax
// rdi == 0x0000000077AB4258 aCreatefilew    db 'CreateFileW',0

```

获取到 rdi 之后  
设置好 rsi 开始

```
lodsb
// rsi: 2FD7F4-> 2FD7F5
// 00000000002FD7F4  9E 13 AB 8B 23 AB B6 CB E3 AB 3E 78 4B 45 52 4E  ..«.#«¶Ëã«>xKERN 
// 00000000002FD804  45 4C 33 32 2E 64 6C 6C 00 00 00 00 02 00 00 00  EL32.dll........
// 这部分空间为vmp开辟的 数据是从vm的bytecode里面获取到的
// 一个获取完后再获取另一个修改该空间数据
// 节省点篇幅 就不贴了

```

```
inc al
xor al,79
dec al
not al
rol al,5
cmp al,byte ptr ds:[rdi]

```

在根据 Export Names 的不同序号 (如: aCreatefilew) 进行比较  
得到想要的 api 的地址 只是这个 api 的 name 是写死在 vm 的字节码中  
然后在写入 vmp0 或者像 LocalAlloc 开辟的空间里  
其实就是我们去获取 api 地址的那些常规逻辑

 

vmp 类似去处理字符串的方式还是很多  
比方说 xjun 师傅的插件绕过 syscall 的处理方式  
就是因为 vmp 就是去读 ntdll .rsrc 里面的版本信息的

 

样本中 处理的 api:

```
INFO: VirtualProtect
INFO: LocalAlloc 0xd0 bytes, allocated at 0x14024d000
INFO: LoadLibraryA KERNEL32.dll, return 7ffdafa30000
INFO: LoadLibraryA NTDLL, return 7ffdafc80000
GET: changeaddr:0x14012CF14 info:0xFFFFFFFF93E98F89 -> 0x7FFD91E75F5E writeaddr:0x140240304
GET: changeaddr:0x1400DC999 info:0x20C54ECF -> 0x7FFD58A80EA0 writeaddr:0x140240304
GET: changeaddr:0x1401386D2 info:0xFFFFFFFF84E4C247 -> 0x7FFDD0C06208 writeaddr:0x140240304
INFO: LoadLibraryA NTDLL, return 7ffdafc80000
GET: changeaddr:0x140114DF1 info:0x72EC197C -> 0x7FFE0021F4CF writeaddr:0x140240304
GET: changeaddr:0x1400BF700 info:0xFFFFFFFFA0F139EB -> 0x7FFDF9B10524 writeaddr:0x140240304
INFO: LoadLibraryA NTDLL, return 7ffdafc80000
GET: changeaddr:0x1400A8424 info:0x6720DAF0 -> 0x7FFD5BBD122B writeaddr:0x140240304
GET: changeaddr:0x1400BF5A1 info:0x5933AA4D -> 0x7FFD8B625FFA writeaddr:0x140240304
GET: changeaddr:0x1400DE9BF info:0x1BAC8109 -> 0x7FFDDA6D076E writeaddr:0x140240304
GET: changeaddr:0x1401058EE info:0xFFFFFFFF856E310C -> 0x7FFD52CBA43F writeaddr:0x140240304
GET: changeaddr:0x1400F80D6 info:0xFFFFFFFF804959BC -> 0x7FFDA608800F writeaddr:0x140240304
GET: changeaddr:0x140131670 info:0xFFFFFFFFD5603F83 -> 0x7FFDC7E3B62C writeaddr:0x140240304
GET: changeaddr:0x1400D8B98 info:0xFFFFFFFFD4849A73 -> 0x7FFDB5C9E14C writeaddr:0x140240304
GET: changeaddr:0x1401150D7 info:0xFFFFFFFFADD9A510 -> 0x7FFD60C0DE7B writeaddr:0x140240304
GET: changeaddr:0x1400B4374 info:0xFFFFFFFFCABBE1F2 -> 0x7FFD31DFFF21 writeaddr:0x140240304
GET: changeaddr:0x1400B6D19 info:0x4A88125B -> 0x7FFD9EA510F4 writeaddr:0x140240304
GET: changeaddr:0x14012D238 info:0xFFFFFFFFBFC07818 -> 0x7FFE2A2E4D83 writeaddr:0x140240304
GET: changeaddr:0x1401372EB info:0x59B6839 -> 0x7FFD600DC55E writeaddr:0x140240304
GET: changeaddr:0x140148E72 info:0xFFFFFFFFC1383704 -> 0x7FFE0D05C2F7 writeaddr:0x140240304
GET: changeaddr:0x1400CBB51 info:0x6E826F15 -> 0x7FFE1D225532 writeaddr:0x140240304
GET: changeaddr:0x14014963B info:0xFFFFFFFFD5BCFD7F -> 0x7FFE243EE940 writeaddr:0x140240304
GET: changeaddr:0x14013A908 info:0xFFFFFFFFA6F9E8FA -> 0x7FFD70CA7549 writeaddr:0x140240304
GET: changeaddr:0x140148F24 info:0x49504912 -> 0x7FFDFACE0F01 writeaddr:0x140240304
GET: changeaddr:0x140112E5B info:0xFFFFFFFF8D4D6BFB -> 0x7FFDC19440F4 writeaddr:0x140240304
GET: changeaddr:0x1400AC101 info:0xFFFFFFFFFB180638 -> 0x7FFD6B2FB7D3 writeaddr:0x140240304
INFO: LoadLibraryA NTDLL, return 7ffdafc80000
GET: changeaddr:0x1400E0B3E info:0x7067D4D9 -> 0x7FFD7D335CFE writeaddr:0x140240304
INFO: LoadLibraryA NTDLL, return 7ffdafc80000
GET: changeaddr:0x1400F47A3 info:0x78C1C94E -> 0x7FFDA5C12755 writeaddr:0x140240304
GET: changeaddr:0x1400A3690 info:0x1454C37B -> 0x7FFD99521A34 writeaddr:0x140240304
GET: changeaddr:0x14009D008 info:0x2EED23B8 -> 0x7FFD98D0C603 writeaddr:0x140240304
GET: changeaddr:0x1400A7261 info:0x4716F059 -> 0x7FFE2574C0CE writeaddr:0x140240304
GET: changeaddr:0x14009D010 info:0xFFFFFFFFFEDF0E91 -> 0x7FFDAD3645F6 writeaddr:0x140240304
GET: changeaddr:0x14013D4B0 info:0xFFFFFFFF9503B9D6 -> 0x7FFD540757CD writeaddr:0x140240304
GET: changeaddr:0x1400F7159 info:0x705B585F -> 0x7FFDA14A7AB0 writeaddr:0x140240304
GET: changeaddr:0x1401218A7 info:0xFFFFFFFF934988BA -> 0x7FFDC6DCF4B9 writeaddr:0x140240304
GET: changeaddr:0x140107F3E info:0xFFFFFFFFB270D741 -> 0x7FFD3A62DAA6 writeaddr:0x140240304
GET: changeaddr:0x1401251B4 info:0xFFFFFFFFE3737767 -> 0x7FFDB992E148 writeaddr:0x140240304
GET: changeaddr:0x1400AB3BF info:0xFFFFFFFF8DFB51C5 -> 0x7FFD95629372 writeaddr:0x140240304
GET: changeaddr:0x1400E397B info:0x5DCFBA53 -> 0x7FFD4BE350BC writeaddr:0x140240304
GET: changeaddr:0x1400F814C info:0xFFFFFFFFCBD2DAA6 -> 0x7FFDEE90566D writeaddr:0x140240304
GET: changeaddr:0x140112A4D info:0x34EFD15 -> 0x7FFD4BEAF4F2 writeaddr:0x140240304
GET: changeaddr:0x1400C1742 info:0x2613FCB2 -> 0x7FFE255D2E71 writeaddr:0x140240304
GET: changeaddr:0x1400A4B2E info:0x63A1EE13 -> 0x7FFE2E194F1C writeaddr:0x140240304
GET: changeaddr:0x140117E5D info:0x4327F530 -> 0x7FFD93E7FD8B writeaddr:0x140240304
GET: changeaddr:0x1400BD1FF info:0xFFFFFFFFA4C2D29C -> 0x7FFD86562C2F writeaddr:0x140240304
GET: changeaddr:0x14014A5D9 info:0x3E024361 -> 0x7FFE23CB6F26 writeaddr:0x140240304
GET: changeaddr:0x1400C8BD8 info:0x1E0DE56 -> 0x7FFE08E4408D writeaddr:0x140240304
GET: changeaddr:0x1400F1360 info:0x2741D30F -> 0x7FFDE207E8F0 writeaddr:0x140240304
GET: changeaddr:0x140147C39 info:0x2DD31705 -> 0x7FFE0A762C12 writeaddr:0x140240304
GET: changeaddr:0x140145B19 info:0x74AEF008 -> 0x7FFD6C2DFA43 writeaddr:0x140240304
GET: changeaddr:0x1400BEB54 info:0x7DB8A260 -> 0x7FFDEC48374B writeaddr:0x140240304
GET: changeaddr:0x1400E758B info:0x3D2F2921 -> 0x7FFDD3735EB6 writeaddr:0x140240304
GET: changeaddr:0x1400E6217 info:0x5A317B3D -> 0x7FFD9F31DA5A writeaddr:0x140240304
GET: changeaddr:0x14014C0BE info:0x5409F81D -> 0x7FFD3693D82A writeaddr:0x140240304
GET: changeaddr:0x1400D89C0 info:0xFFFFFFFFAFB576C6 -> 0x7FFDC1A2C92D writeaddr:0x140240304
GET: changeaddr:0x1400FE35B info:0xFFFFFFFF9EA377CC -> 0x7FFE2646016F writeaddr:0x140240304
GET: changeaddr:0x1400C6689 info:0xFFFFFFFF90243DAF -> 0x7FFDC1A13E30 writeaddr:0x140240304
GET: changeaddr:0x1400DC743 info:0x1D7D1527 -> 0x7FFDB40AADD8 writeaddr:0x140240304
GET: changeaddr:0x1400E0B0C info:0x3CC10F6E -> 0x7FFE00722E95 writeaddr:0x140240304
GET: changeaddr:0x1400F0AFA info:0xFFFFFFFF856FFA1B -> 0x7FFDA69EE984 writeaddr:0x140240304
GET: changeaddr:0x1400F6B85 info:0x560381E2 -> 0x7FFE0B833321 writeaddr:0x140240304
GET: changeaddr:0x140148991 info:0xFFFFFFFFF0D6B4FA -> 0x7FFDC64A18F9 writeaddr:0x140240304
GET: changeaddr:0x1400AA7B1 info:0x16D0C303 -> 0x7FFD60E2F5DC writeaddr:0x140240304
GET: changeaddr:0x14012E09B info:0x24FA822A -> 0x7FFD624B2549 writeaddr:0x140240304
GET: changeaddr:0x1400CE7E6 info:0xFFFFFFFFA9BC3DF3 -> 0x7FFDBF66DA8C writeaddr:0x140240304
GET: changeaddr:0x140106D06 info:0xFFFFFFFFD22E7E90 -> 0x7FFE1EF2A2DB writeaddr:0x140240304
GET: changeaddr:0x140106536 info:0x63B7BF11 -> 0x7FFD75F9DCE6 writeaddr:0x140240304
GET: changeaddr:0x140128865 info:0x2CE579C6 -> 0x7FFD7627DCED writeaddr:0x140240304
GET: changeaddr:0x14014A055 info:0xFFFFFFFFF2D0013F -> 0x7FFDA6F06B50 writeaddr:0x140240304
GET: changeaddr:0x1401230D3 info:0xFFFFFFFFF9B6C6B7 -> 0x7FFD9ADD1E08 writeaddr:0x140240304
GET: changeaddr:0x140126685 info:0xFFFFFFFF92E24F24 -> 0x7FFD7F42C7C7 writeaddr:0x140240304
GET: changeaddr:0x14010E102 info:0x2A256DB5 -> 0x7FFD72A8E302 writeaddr:0x140240304
GET: changeaddr:0x1401029A8 info:0xFFFFFFFFB14421BA -> 0x7FFD3214C7B9 writeaddr:0x140240304
GET: changeaddr:0x1400A4D39 info:0xFFFFFFFFB6D3E9D2 -> 0x7FFDF258D9E1 writeaddr:0x140240304
GET: changeaddr:0x140106556 info:0x415B3BB -> 0x7FFDECABD944 writeaddr:0x140240304
GET: changeaddr:0x1400BA557 info:0xFFFFFFFFAB770102 -> 0x7FFDD9B4E0F1 writeaddr:0x140240304
GET: changeaddr:0x1401231B8 info:0xFFFFFFFFAB1B80AB -> 0x7FFDC4652F14 writeaddr:0x140240304
GET: changeaddr:0x1400DDED6 info:0xFFFFFFFF85B54C32 -> 0x7FFD9E390E51 writeaddr:0x140240304
GET: changeaddr:0x1400DCA31 info:0xFFFFFFFFEAC3119B -> 0x7FFE17E11764 writeaddr:0x140240304
GET: changeaddr:0x1400D7EAD info:0x7D6F5058 -> 0x7FFD8C424163 writeaddr:0x140240304
GET: changeaddr:0x140120C23 info:0x68A62D79 -> 0x7FFDB9239C2E writeaddr:0x140240304
GET: changeaddr:0x1400B414F info:0xFFFFFFFFBB3767B1 -> 0x7FFDA8889926 writeaddr:0x140240304
GET: changeaddr:0x14009FBD0 info:0x6D6F62E6 -> 0x7FFDCE41625D writeaddr:0x140240304
GET: changeaddr:0x1400FC946 info:0x142944DF -> 0x7FFDDC539490 writeaddr:0x140240304
GET: changeaddr:0x1400B48FE info:0x45E75657 -> 0x7FFD96DB4DB8 writeaddr:0x140240304
GET: changeaddr:0x14010384A info:0xFFFFFFFFF7AC0B44 -> 0x7FFDA1520E67 writeaddr:0x140240304
GET: changeaddr:0x1400BED73 info:0xFFFFFFFF897BBC47 -> 0x7FFE10E5BF88 writeaddr:0x140240304
GET: changeaddr:0x14014B947 info:0xFFFFFFFFEE79D0BF -> 0x7FFDB1A35660 writeaddr:0x140240304
GET: changeaddr:0x140120E42 info:0xFFFFFFFFC14B7576 -> 0x7FFD9CE2C74D writeaddr:0x140240304
GET: changeaddr:0x1400C3234 info:0xFFFFFFFFA8F9FD71 -> 0x7FFE0084B926 writeaddr:0x140240304
GET: changeaddr:0x14011D02C info:0xFFFFFFFF8DF488A9 -> 0x7FFD3E1750BE writeaddr:0x140240304
GET: changeaddr:0x140110E71 info:0x3D5C363E -> 0x7FFD46E7E325 writeaddr:0x140240304
GET: changeaddr:0x1400A65BC info:0xFFFFFFFFAEFA0617 -> 0x7FFD68DF5598 writeaddr:0x140240304
GET: changeaddr:0x14013E3D3 info:0xFFFFFFFFC432726E -> 0x7FFDCC2F3C55 writeaddr:0x140240304
GET: changeaddr:0x14011B897 info:0xFFFFFFFF8E47DB13 -> 0x7FFD62A7A37C writeaddr:0x140240304
GET: changeaddr:0x1400C3014 info:0xFFFFFFFF9CE895AD -> 0x7FFD947CC89A writeaddr:0x140240304
GET: changeaddr:0x140109835 info:0x6A106812 -> 0x7FFD4F117D11 writeaddr:0x140240304
GET: changeaddr:0x14009E95C info:0xFFFFFFFFADB3D6FB -> 0x7FFDF86848B4 writeaddr:0x140240304
INFO: LoadLibraryA NTDLL, return 7ffdafc80000
GET: changeaddr:0x1400DE1FD info:0xFFFFFFFFF2200CF3 -> 0x7FFD66E119CC writeaddr:0x140240304
INFO: LoadLibraryA NTDLL, return 7ffdafc80000
GET: changeaddr:0x1400EE25A info:0x6BACD3EB -> 0x7FFD78C7F5A4 writeaddr:0x140240304
GET: changeaddr:0x1400F0906 info:0x22A28AE8 -> 0x7FFD938FFC53 writeaddr:0x140240304
GET: changeaddr:0x1400F7FD7 info:0x73182140 -> 0x7FFDECFF4C1B writeaddr:0x140240304
GET: changeaddr:0x1400D8EFC info:0xFFFFFFFF84286D81 -> 0x7FFDC6544146 writeaddr:0x140240304
INFO: LoadLibraryA NTDLL, return 7ffdafc80000
GET: changeaddr:0x14013BE6D info:0xFFFFFFFFB7F396F6 -> 0x7FFE0E786AED writeaddr:0x140240304
GET: changeaddr:0x1400D0759 info:0x4189DF8E -> 0x7FFDDC361815 writeaddr:0x140240304
GET: changeaddr:0x1401200E3 info:0xFFFFFFFFC127ECA0 -> 0x7FFDE6D233FB writeaddr:0x140240304
GET: changeaddr:0x1400EC169 info:0xFFFFFFFFF1D18861 -> 0x7FFD992B9656 writeaddr:0x140240304
GET: changeaddr:0x1400D27D9 info:0x14AA2756 -> 0x7FFD9951911D writeaddr:0x140240304
GET: changeaddr:0x140109A01 info:0xFFFFFFFFF14DF00F -> 0x7FFDC8D6ECB0 writeaddr:0x140240304
GET: changeaddr:0x1400F24BC info:0xFFFFFFFFCE6C0800 -> 0x7FFDAB60C82B writeaddr:0x140240304
GET: changeaddr:0x140146CA1 info:0xFFFFFFFF834F3341 -> 0x7FFD9E9B6A66 writeaddr:0x140240304
GET: changeaddr:0x1400A20C8 info:0x426287B6 -> 0x7FFDA0068E2D writeaddr:0x140240304
INFO: LoadLibraryA NTDLL, return 7ffdafc80000
GET: changeaddr:0x1400C4082 info:0x438F3EB1 -> 0x7FFDFC1A3AB6 writeaddr:0x140240304
GET: changeaddr:0x14012FB51 info:0x60DD21E9 -> 0x7FFD6686ECEE writeaddr:0x140240304
GET: changeaddr:0x14009D000 info:0xFFFFFFFFFD596E21 -> 0x7FFE1F0CDF56 writeaddr:0x140240304
INFO: LoadLibraryA USER32.dll, return 7ffdadf10000
GET: changeaddr:0x1400D9A08 info:0x6D383B47 -> 0x7FFD730E9E58 writeaddr:0x140240304
GET: changeaddr:0x14010C195 info:0xFFFFFFFFFB34B816 -> 0x7FFE1E48AEAD writeaddr:0x140240304
GET: changeaddr:0x1400FC262 info:0xFFFFFFFFC0EF9CAE -> 0x7FFDFF035815 writeaddr:0x140240304
GET: changeaddr:0x1400E8392 info:0xF1570F4 -> 0x7FFE24A1A657 writeaddr:0x140240304
GET: changeaddr:0x140141F22 info:0x34892C5B -> 0x7FFDA9868F44 writeaddr:0x140240304
GET: changeaddr:0x1400EFBE6 info:0xFFFFFFFF89BADA18 -> 0x7FFE0C0F65D3 writeaddr:0x140240304
GET: changeaddr:0x1400CF338 info:0xFFFFFFFF8FBED14B -> 0x7FFD90EEBCD4 writeaddr:0x140240304
GET: changeaddr:0x140127915 info:0x677FFFC8 -> 0x7FFD6246B703 writeaddr:0x140240304
GET: changeaddr:0x1400D2B35 info:0xFFFFFFFFC5347FA9 -> 0x7FFD74C966AE writeaddr:0x140240304
GET: changeaddr:0x1400BEA2B info:0x725334D7 -> 0x7FFE24530E78 writeaddr:0x140240304
GET: changeaddr:0x1400EBD79 info:0x50C7DBF3 -> 0x7FFD8D014B5C writeaddr:0x140240304
GET: changeaddr:0x1401475D2 info:0xFFFFFFFFD276974F -> 0x7FFD311DA750 writeaddr:0x140240304
GET: changeaddr:0x140138130 info:0xDA17FC6 -> 0x7FFDEDB2229D writeaddr:0x140240304
GET: changeaddr:0x1400A81F6 info:0x6DFD3F3F -> 0x7FFDF9648920 writeaddr:0x140240304
GET: changeaddr:0x1400AC3E8 info:0xFFFFFFFF939044B7 -> 0x7FFD82F1EB88 writeaddr:0x140240304
GET: changeaddr:0x1400E43BE info:0xFFFFFFFFE21CFBB5 -> 0x7FFDC3E975B2 writeaddr:0x140240304
GET: changeaddr:0x14012EBF5 info:0x26C487BA -> 0x7FFD80491799 writeaddr:0x140240304
GET: changeaddr:0x1400B7EEB info:0xFFFFFFFF945564C3 -> 0x7FFD7A321E6C writeaddr:0x140240304
GET: changeaddr:0x140114149 info:0xFFFFFFFF93E187A0 -> 0x7FFD3B93DCAB writeaddr:0x140240304
GET: changeaddr:0x14011CC23 info:0xFFFFFFFFDCC116DD -> 0x7FFE2771F3BA writeaddr:0x140240304
GET: changeaddr:0x1400DFE6A info:0x14B0CB50 -> 0x7FFD3B22652B writeaddr:0x140240304
GET: changeaddr:0x1400D2E6D info:0x37B9A2D1 -> 0x7FFD38CEE8E6 writeaddr:0x140240304
GET: changeaddr:0x1400DCB00 info:0x7B2F7B4D -> 0x7FFD7FFFFBCA writeaddr:0x140240304
GET: changeaddr:0x140125210 info:0xFFFFFFFFA4B39B79 -> 0x7FFD2E772E7E writeaddr:0x140240304
GET: changeaddr:0x140142C13 info:0xFFFFFFFFEDDB2883 -> 0x7FFDA3A7005C writeaddr:0x140240304
GET: changeaddr:0x1400DA17C info:0x649ADD7B -> 0x7FFE10D85114 writeaddr:0x140240304
GET: changeaddr:0x1400F92BC info:0x53085B8 -> 0x7FFD99462163 writeaddr:0x140240304
GET: changeaddr:0x1400E30A2 info:0x2DCA3A59 -> 0x7FFDE0196A0E writeaddr:0x140240304
GET: changeaddr:0x14013FE71 info:0xFFFFFFFF92851891 -> 0x7FFDDC1CFD26 writeaddr:0x140240304
GET: changeaddr:0x14013CEF0 info:0x8E0FFC9 -> 0x7FFD2F5AFA8E writeaddr:0x140240304
GET: changeaddr:0x140139F67 info:0xFFFFFFFFF45CDFDE -> 0x7FFE0DBA9D25 writeaddr:0x140240304
GET: changeaddr:0x14013CC45 info:0x2B99FB76 -> 0x7FFDDA8EB0CD writeaddr:0x140240304
GET: changeaddr:0x14011395F info:0x1EE0DAAF -> 0x7FFDCD4D2410 writeaddr:0x140240304
GET: changeaddr:0x14010BFB1 info:0x577D717C -> 0x7FFDD612C20F writeaddr:0x140240304
GET: changeaddr:0x1400E6E32 info:0xFFFFFFFFF4D0FAC8 -> 0x7FFE0645A003 writeaddr:0x140240304
GET: changeaddr:0x1400DB0E5 info:0xFFFFFFFFC0F9F952 -> 0x7FFDA1EA5F21 writeaddr:0x140240304
INFO: LoadLibraryA ADVAPI32.dll, return 7ffdad4a0000
GET: changeaddr:0x1400B2C6C info:0xFFFFFFFF8A46E93B -> 0x7FFD3FC5F1C4 writeaddr:0x140240304
GET: changeaddr:0x14010895D info:0x4E246015 -> 0x7FFDB45A2DB2 writeaddr:0x140240304
GET: changeaddr:0x140136F07 info:0x7FF2941D -> 0x7FFD883F05EA writeaddr:0x140240304
GET: changeaddr:0x1400BCED4 info:0x9F711D5 -> 0x7FFDD795B2F2 writeaddr:0x140240304
GET: changeaddr:0x1400F2A75 info:0xFFFFFFFF9DE2315E -> 0x7FFD73738FA5 writeaddr:0x140240304
GET: changeaddr:0x1400EB284 info:0x4C4303B7 -> 0x7FFDC4518A68 writeaddr:0x140240304
GET: changeaddr:0x1400E7E39 info:0xFFFFFFFF86C3CA2F -> 0x7FFD45B7B2A0 writeaddr:0x140240304
GET: changeaddr:0x14013E3DB info:0xFFFFFFFFB9C906FC -> 0x7FFD44DBD82F writeaddr:0x140240304
INFO: LoadLibraryA SHELL32.dll, return 142300000
GET: changeaddr:0x14014ACDC info:0x71E27D6D -> 0xE4BBC77A writeaddr:0x140240304
INFO: LoadLibraryA ole32.dll, return 7ffdad630000
INFO: LoadLibraryA api-ms-win-core-com-l1-1-0, return 7ffdad200000
GET: changeaddr:0x1400C5013 info:0xFFFFFFFF9D25EE4A -> 0x7FFD4235A7C9 writeaddr:0x140240304
GET: changeaddr:0x1400EDD0A info:0xFFFFFFFFEBF562D2 -> 0x7FFDDC0C9811 writeaddr:0x140240304
INFO: LoadLibraryA api-ms-win-core-com-l1-1-0, return 7ffdad200000
GET: changeaddr:0x1400DD1DF info:0x236917EA -> 0x7FFDB46B7179 writeaddr:0x140240304
INFO: LoadLibraryA api-ms-win-core-com-l1-1-0, return 7ffdad200000
GET: changeaddr:0x140128103 info:0xFFFFFFFFF06A3A02 -> 0x7FFD4D0FED31 writeaddr:0x140240304
INFO: LoadLibraryA api-ms-win-core-com-l1-1-0, return 7ffdad200000
GET: changeaddr:0x1400E3316 info:0x61D64DAB -> 0x7FFD768B9E34 writeaddr:0x140240304
INFO: LoadLibraryA api-ms-win-core-com-l1-1-0, return 7ffdad200000
GET: changeaddr:0x1400D5978 info:0x4AD9BA05 -> 0x7FFDF2D00B12 writeaddr:0x140240304
INFO: LoadLibraryA OLEAUT32.dll, return 7ffdae1e0000
GET: changeaddr:0x140106198 info:0xFFFFFFFF99A5A093 -> 0x7FFDA276E06C writeaddr:0x140240304
INFO: LoadLibraryA SHLWAPI.dll, return 7ffdad1a0000
GET: changeaddr:0x1400B3A18 info:0xFFFFFFFFF1D2B1BC -> 0x7FFDDA6FE89F writeaddr:0x140240304
GET: changeaddr:0x14013AAFB info:0x145BE9B0 -> 0x7FFE0D47F58B writeaddr:0x140240304
GET: changeaddr:0x140117D1E info:0xFFFFFFFFEFCDA960 -> 0x7FFD9FB8697B writeaddr:0x140240304
GET: changeaddr:0x1400A1783 info:0xFFFFFFFFF39C62AE -> 0x7FFDBAB2A895 writeaddr:0x140240304
INFO: LoadLibraryA VERSION.dll, return 7ffda0730000
GET: changeaddr:0x1400DBC54 info:0x1B0764C0 -> 0x7FFE1B5294AB writeaddr:0x140240304
GET: changeaddr:0x1400FD105 info:0xFFFFFFFFB58675DA -> 0x7FFD3935D719 writeaddr:0x140240304
GET: changeaddr:0x14012D421 info:0x4827B163 -> 0x7FFE1515093C writeaddr:0x140240304
INFO: LoadLibraryA GDI32.dll, return 7ffdadd80000
GET: changeaddr:0x14013A87E info:0x4816701 -> 0x7FFD9625B3B6 writeaddr:0x140240304
GET: changeaddr:0x1400DACCF info:0x619170ED -> 0x7FFDD058129A writeaddr:0x140240304
GET: changeaddr:0x1401053C8 info:0x45FC15CD -> 0x7FFD36418B7A writeaddr:0x140240304
GET: changeaddr:0x140147F22 info:0x91D7D85 -> 0x7FFD2F5D0D12 writeaddr:0x140240304
GET: changeaddr:0x1400FAB20 info:0xFFFFFFFFCC633864 -> 0x7FFD9BADA247 writeaddr:0x140240304
GET: changeaddr:0x1400E4865 info:0xFFFFFFFFEC1AF0E7 -> 0x7FFD65DDBF78 writeaddr:0x140240304
GET: changeaddr:0x140133345 info:0xFFFFFFFFB0BA47FE -> 0x7FFDF5212A85 writeaddr:0x140240304
GET: changeaddr:0x14013F22E info:0x66918CD9 -> 0x7FFD6994B52E writeaddr:0x140240304
GET: changeaddr:0x1400BBDB7 info:0x6700142E -> 0x7FFD86F99B55 writeaddr:0x140240304
GET: changeaddr:0x14010BAB5 info:0xFFFFFFFFD9F0F8C7 -> 0x7FFE25A01B28 writeaddr:0x140240304
GET: changeaddr:0x1400F7C0E info:0xFFFFFFFFE8E4AC74 -> 0x7FFE2CCE1B07 writeaddr:0x140240304
GET: changeaddr:0x1400D5D87 info:0x3779C2B7 -> 0x7FFDFA522518 writeaddr:0x140240304
GET: changeaddr:0x140117FD5 info:0x2EC07B24 -> 0x7FFD8370D087 writeaddr:0x140240304
GET: changeaddr:0x1400A564F info:0x3A489B5 -> 0x7FFDC6F1B7A2 writeaddr:0x140240304
GET: changeaddr:0x1400D0098 info:0xFFFFFFFFC4B9646D -> 0x7FFDAB0455FA writeaddr:0x140240304
GET: changeaddr:0x140128A11 info:0x39A2B825 -> 0x7FFDB58B1B42 writeaddr:0x140240304
GET: changeaddr:0x1400C248E info:0xFFFFFFFFCDB764DD -> 0x7FFE0252CB0A writeaddr:0x140240304
GET: changeaddr:0x1400F25C8 info:0xFFFFFFFF93234D02 -> 0x7FFE1A5C4011 writeaddr:0x140240304
GET: changeaddr:0x1400EB787 info:0xFFFFFFFFE2473CAB -> 0x7FFD41CD5A54 writeaddr:0x140240304
GET: changeaddr:0x140139362 info:0xFFFFFFFFDE223AA8 -> 0x7FFD93A7FAB3 writeaddr:0x140240304
GET: changeaddr:0x1400A5799 info:0xFFFFFFFFC97EB009 -> 0x7FFD9B4B2E3E writeaddr:0x140240304
GET: changeaddr:0x1400C712B info:0xFFFFFFFF8BDA5D1E -> 0x7FFD7999EE65 writeaddr:0x140240304
INFO: VirtualProtect

```

以上过程就是 vmp 在 LocalAlloc 之后  
LoadLibrary dll 修复完 vmp0 中各个 api 的地址的过程  
LocalAlloc 中的数据填写也类似 此外还有 cpuid 的运算

 

btw:  
明白了 就能去处理这些东西了  
思路会有一些 不同思路会处理起来不一样  
一些思路上一些需要注意的点  
1.  
vmp1 里面会调用到一些 api，涉及到空间问题  
2.  
0xAAA: call 0xBBB  
0xBBB: call 0xCCC // CCC 去解密调用 api  
这种需要将 BBB 修复成 jmp  
3.  
反汇编引擎 search 0xe8 会得到两种 // 感谢大表哥 (hzqst) 提供的暴搜 0xe8 以及开源的 unicorn_pe  
一种为 5 字节的  
一种为 6 字节的 即 opcode 中会包含前缀 // 前面如果会有 push reg opcode 也会有前缀 即 2 字节  
4.  
根据堆栈情况  
call/jmp  
前移 / 补 ret

 

还有的 等我遇到再补充

[【公告】看雪招聘大学实习生！看雪 20 年安全圈的口碑，助你快速成长！](https://job.kanxue.com/position-read-1104.htm)