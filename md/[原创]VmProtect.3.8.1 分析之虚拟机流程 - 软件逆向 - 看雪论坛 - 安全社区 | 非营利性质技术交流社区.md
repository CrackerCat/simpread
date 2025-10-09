> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-288666.htm)

> [原创]VmProtect.3.8.1 分析之虚拟机流程

上一篇我分析了 VmProtect.3.0.0 的例子，然后这篇我将为大家带来 VmProtect.3.8.1 虚拟机流程分析的例子，有的小伙伴就纳闷了，这跳级也跳的太快了，VmProtect.3.0.0 版本的所有知识你弄懂了没，当然没有，比如什么是指令变异，反调试，内存保护等，这些还是需要大量的时间去研究，有的人喜欢把他们全都弄懂在去研究下个版本，但是这不是我的风格。为啥要选择 VmProtect.3.8.1 版本呢，我能够找到的版本里面 VmProtect.3.8.1 版本选项里面开始多了好多个选项，其中开头的那三个，一个是版本选择框，有两个选项分别是 VMProtect 2.X 和 默认，第二个是 Instances（实例），这个我看不出是什么东西，第三个是 Complexity(复杂程序)，默认是填了 20%。所以说这个版本和之前的比起来又进行了稍微大点的升级。OK，我们进入正题。

这次给大家的附件依旧没有虚拟机主程序，然后我说下，设置啥的，首先加密的是入口点后面的的下面两行代码，和上篇帖子一样

```
0040A5A8 >  B8 AAAAAAAA     mov eax,0xAAAAAAAA
0040A5AD    BB BBBBBBBB     mov ebx,0xBBBBBBBB
0040A5B2    B9 CCCCCCCC     mov ecx,0xCCCCCCCC
0040A5B7    BA DDDDDDDD     mov edx,0xDDDDDDDD
0040A5BC    BE 51515151     mov esi,0x51515151
0040A5C1    BF D1D1D1D1     mov edi,0xD1D1D1D1
0040A5C6    B8 78563412     mov eax,0x12345678 ;
0040A5CB    E9 65AA0000     jmp 123.00415035
0040A5D0    C3              retn
0040A5D1    90              nop

```

加密的是 0040A5C6 和 0040A5CB 这两条代码，然后选项那边，把是都改成否，英文版的，yes 全改 no，然后前面我讲到的多出来的三个设置不动, 最后 Compilation Type(编译类型)，这边选择 Virtualization（虚拟化）。我们把编译成功的 123.vmp.exe 拖到 OD 上

```
0040A5C6   .- E9 22940800   jmp 123_vmp.004939ED
0040A5CB   .  E8 78400900   call 123_vmp.0049E648

```

0040A5C6 这边的代码被替换成了 jmp 指令，0040A5CB 这边的代码可能是被替换成某个 handler 的一部分了吧。我们接着往下单步

```
004939ED    9C              pushfd
;存放标志寄存器eflags到堆栈，后面会用到
[0019FF70] = 00000246
004939EE    52              push edx
;压入edx,[0019FF6C] = DDDDDDDD
004939EF    BA 18B28448     mov edx,0x4884B218
004939F4    F7C2 063C3D78   test edx,0x783D3C06
;这条指令执行完标志寄存器发生变化
004939FA    0f96c2          setbe dl
004939FD    FFB414 044E7BB7 push dword ptr ss:[esp+edx-0x4884B1FC]
;相当于push dword ptr ss:[esp+4]
;那么这个esp+4的4在汇编指令的结构中叫做Displacement
;我们简称DSP，这里其实就是对dsp进行加密了
00493A04    9D              popfd
;把[0019FF70]的值赋给eflags，eflags值变回来了。
00493A05    C74424 04 6330E>mov dword ptr ss:[esp+0x4],0x33E43063
;执行完这条指令，[0019FF70]=0x33E43063,这个值就是待解密的伪代码地址
00493A0D    8B9414 004E7BB7 mov edx,dword ptr ss:[esp+edx-0x4884B200>
;相当于mov edx,dword ptr ss:[esp + 0]
;这里dsp = 0，但是加密后我们无法直接看出来
;只能运行到这条指令才看的出来
00493A14    8D6424 04       lea esp,dword ptr ss:[esp+0x4]
;相当于add esp,0x4
00493A18    E8 DCC30500     call 123_vmp.004EFDF9

```

```
004EFDF9    E8 D8EEF4FF     call 123_vmp.0043ECD6

```

```
0043ECD6   /E9 DD130100     jmp 123_vmp.004500B8
;前面两个call,后面一个jmp，目的就是让代码更加碎片化

```

```
004500B8    890C24          mov dword ptr ss:[esp],ecx
;执行完 [0019FF68] = CCCCCCCC
;这一句就是实现压入ecx到堆栈，push ecx的功能了
004500BB    B9 3C879EBD     mov ecx,0xBD9E873C  ;垃圾指令
004500C0    0FC9            bswap ecx;垃圾指令
004500C2    9C              pushfd
;压入eflags, [0019FF64]=00000246
004500C3    0BC9            or ecx,ecx;垃圾指令
004500C5    F7D1            not ecx ;垃圾指令
004500C7    0F81 71380400   jno 123_vmp.0049393E
;这个相当于jmp，永远都是跳转

```

```
0049393E    B9 00000000     mov ecx,0x0
;重定位值 0 ，实现push 0 的功能，这里是先保存0
;后面再push ecx，这里有很多这样的操作
00493943  ^ 0F83 4F03FDFF   jnb 123_vmp.00463C98
;这个相当于jmp，永远都是跳转

```

```
00463C98    68 B4D0010A     push 0xA01D0B4 
00463C9D    893C24          mov dword ptr ss:[esp],edi
;00463C98的指令加上这条实现了压入 edi的功能
;[0019FF60] = D1D1D1D1
00463CA0    BF 3C462051     mov edi,0x5120463C
00463CA5    52              push edx
;压入edx , [0019FF5C] = DDDDDDDD
00463CA6    57              push edi
00463CA7    89B43C C4B9DFAE mov dword ptr ss:[esp+edi-0x5120463C],esi
;00463CA6的指令加上这条实现了压入 esi的功能
;[0019FF58 ]=51515151
00463CAE    13FF            adc edi,edi
00463CB0    0F8F 892E0600   jg 123_vmp.004C6B3F
;这个相当于jmp，永远都是跳转

```

```
004C6B3F    c1cf 3a         ror edi,0x3a
004C6B42    51              push ecx
;压入重定位值 0 ，[0019FF54] = 0
004C6B43    BE 1D71A355     mov esi,0x55A3711D
004C6B48    8DB434 D78E5CAA lea esi,dword ptr ss:[esp+esi-0x55A37129]
;等效于lea esi，dword ptr ss:[esp-0xC]
004C6B4F  ^ 0F82 A22BFDFF   jb 123_vmp.004996F7
;这个相当于jmp，永远都是跳转

```

```
004996F7    8D947F 1CB41BBF lea edx,dword ptr ds:[edi+edi*2-0x40E44BE4]; 垃圾指令
004996FE    2BD2            sub edx,edx
;相当于 mov edx，0
00499700    55              push ebp
;压入ebp  [0019FF50 ] = 0019FF80
00499701    E8 6A55FFFF     call 123_vmp.0048EC70

```

```
0048EC70    8D9496 38FFFFFF lea edx,dword ptr ds:[esi+edx*4-0xC8]
;等效于lea edx，dword ptr ss:[esi-0xC8]
;这条指令是参与实现sub esp，0xc8的
0048EC77    5F              pop edi
; 这条指令执行完 edi = 00499706。
;这个值也就是执行完00499701的call后压入下一条指令的地址    
0048EC78    81D7 843E0000   adc edi,0x3E84
;算出下一条代码块的地址，为什么是下一条代码块
;而不是handler呢，我们可以很轻松看出来堆栈上
;并没有eax和ebx的值，当然还有其他初始化没完成
0048EC7E  - FFE7            jmp edi   ;去执行下一条代码块

```

这条代码块根据以前经验，执行完应该把相应的寄存器都入栈了还有执行其他初始化了的，但是这里并没有这样做。只入栈了部分寄存器，根据分析下一条代码块，得出结论，它被拆分成了两条代码块用 jmp edi 相连，共同完成压入寄存器和一些初始化的操作。我们看下一个代码块

```
0049D58A    E8 9192FCFF     call 123_vmp.00466820

```

```
00466820    891C24          mov dword ptr ss:[esp],ebx
;实现压入ebx的功能了，[0019FF4C] = BBBBBBBB
00466823    8B7C24 24       mov edi,dword ptr ss:[esp+0x24]
;这里给edi赋的值是待解密的伪代码地址
00466827    BB 394DB212     mov ebx,0x12B24D39
0046682C    F7DF            neg edi  ;解密伪代码地址                           
0046682E    E9 F0070900     jmp 123_vmp.004F7023

```

```
004F7023    0FBEEB          movsx ebp,bl;垃圾指令
004F7026    81EF 85022AC5   sub edi,0xC52A0285 ;解密伪代码地址
004F702C    D1C7            rol edi,1;解密伪代码地址
004F702E    47              inc edi ;解密伪代码地址
004F702F    E8 52B9FBFF     call 123_vmp.004B2986

```

```
004B2986    E8 85740000     call 123_vmp.004B9E10

```

```
004B9E10    81F7 81EDA10D   xor edi,0xDA1ED81;解密伪代码地址
004B9E16    E8 92CFF9FF     call 123_vmp.00456DAD

```

```
00456DAD    89841C CFB24DED mov dword ptr ss:[esp+ebx-0x12B24D31],eax
;相当于mov dword ptr ss:[esp+8],eax
;这边就完成了压入eax的功能，[0019FF48] = AAAAAAAA
00456DB4    899C9C 1CCB36B5 mov dword ptr ss:[esp+ebx*4-0x4AC934E4],ebx
;相当于mov dword ptr ss:[esp],ebx
;[0019FF40] = 12B24D39
00456DBB    66:c1f5 41      sal bp,0x41  ;垃圾指令
00456DBF    13F9            adc edi,ecx
;伪代码地址解密计算完成
00456DC1    E8 D7520100     call 123_vmp.0046C09D

```

```
0046C09D    8BE2            mov esp,edx
;这里配合0048EC70的代码，相当于sub esp,0xC8
0046C09F    BD 9A1BBA01     mov ebp,0x1BA1B9A
0046C0A4    66:c1ed 27      shr bp,0x27
0046C0A8    8BDF            mov ebx,edi
; 伪代码地址给ebx
;ebx参与我们上一篇帖子讲的双重解密
0046C0AA  ^ 0F85 6028FCFF   jnz 123_vmp.0042E910

```

```
0042E910    B8 00000000     mov eax,0x0 ;垃圾指令
0042E915    81D5 38629BFE   adc ebp,0xFE9B6238 ;垃圾指令
0042E91B    2BD8            sub ebx,eax ;垃圾指令
0042E91D    B9 1AC72B79     mov ecx,0x792BC71A
0042E922    8BC1            mov eax,ecx ;垃圾指令
0042E924    8D04CD 9443A743 lea eax,dword ptr ds:[ecx*8+0x43A74394]
;相当于 mov eax，0x0D057C64
0042E92B    8D2D 1DE94200   lea ebp,dword ptr ds:[0x42E91D]
相当于 mov ebp ，0x42E91D
;这个值参与计算下一条handler
0042E931    c1f9 33         sar ecx,0x33
0042E934    F7D0            not eax        ;垃圾指令
0042E936    8B848F 6CC3FFFF mov eax,dword ptr ds:[edi+ecx*4-0x3C94]
;等效于mov eax,dword ptr ds:[edi]
;这边功能就是读取伪代码4个字节到eax
0042E93D    8D144D 3352A749 lea edx,dword ptr ds:[ecx*2+0x49A75233]
;相当于 mov edx，0x49A7707D
0042E944    0F83 D8070900   jnb 123_vmp.004BF122
;这个相当于jmp，永远都是跳转

```

```
004BF122    c0c9 61         ror cl,0x61 
004BF125    8DBC39 72F0FFFF lea edi,dword ptr ds:[ecx+edi-0xF8E]
;等效于add edi,0x4
004BF12C    c1e9 6d         shr ecx,0x6d
;相当于mov ecx，0
004BF12F    66:0BD2         or dx,dx    
004BF132    33C3            xor eax,ebx  ;开始解密eax,ebx作为密钥
004BF134    c0e2 85         shl dl,0x85 
004BF137    8D8401 8BADAB7E lea eax,dword ptr ds:[ecx+eax+0x7EABAD8B]
;等效于add eax，0x7EABAD8B,解密eax
004BF13E    F6D2            not dl 
004BF140    F7D0            not eax;解密eax
004BF142    66:81CA B81D    or dx,0x1DB8  
004BF147    c1ea 7b         shr edx,0x7b 
004BF14A    C0C2 05         rol dl,0x5 
004BF14D    35 15C82EEB     xor eax,0xEB2EC815 ;解密eax
004BF152    c1f2 d3         sal edx,0xd3  
004BF155    2AD6            sub dl,dh  
004BF157    c1c2 2c         rol edx,0x2c 
004BF15A    48              dec eax      ;解密eax
004BF15B    F7D0            not eax      ;解密eax完成
004BF15D    0FC0E9          xadd cl,ch  
004BF160    33D8            xor ebx,eax ;生成下次解密的密钥
004BF162    86D5            xchg ch,dl  
004BF164    33CA            xor ecx,edx 
004BF166    E8 19970100     call 123_vmp.004D8884

```

```
004D8884    13E8            adc ebp,eax
; ebp的值来源于0042E92B的指令
;算出下一条handler的地址
004D8886    0FB7C1          movzx eax,cx
004D8889    898C14 00000080 mov dword ptr ss:[esp+edx-0x80000000],ecx
;等效于mov dword ptr ss:[esp],ecx
;[0019FE7C] =  80001000
;这个值后面没有用到，所以这条是垃圾指令
004D8890    52              push edx
004D8891    89AC14 00000080 mov dword ptr ss:[esp+edx-0x80000000],ebp
004D8898    C2 0400         retn 0x4 ;配合上面两条代码去执行handler

```

分析到这里我们推断出某些寄存器对应的作用，ebx 是用来存放密钥的，esp 是存放虚拟寄存器的基地址的，ebp 相当于 3.0.0 版本的 edi，用于计算 handler 的地址，esi 相当于 3.0.0 版本的 ebp, 用于数据的存取和计算，虚拟机中 ebp 的作用和真实指令的 esp 作用差不多，最后这个 edi 相当于 3.0.0 版本的 esi，用于存放伪代码的指针。这个版本的垃圾指令并不多，很多看似是像的其实是参与解密 DSP 用的。  
这里我们可以看到 esi 指向的堆栈对应寄存器分布

```
0019FF48   AAAAAAAA ;eax
0019FF4C   BBBBBBBB ;ebx
0019FF50   0019FF80 ;ebp
0019FF54   00000000 ;relocation(重定位值 0)
0019FF58   51515151 ;esi
0019FF5C   DDDDDDDD ;edx
0019FF60   D1D1D1D1 ;edi
0019FF64   00000246 ;eflags
0019FF68   CCCCCCCC ;ecx

```

至此所有寄存器的入栈，和其他的初始化完成了，下面就是去执行 handler 了，

```
004A107C    B8 35F78550     mov eax,0x5085F735
004A1081    8B8C30 CB087AAF mov ecx,dword ptr ds:[eax+esi-0x5085F735]
;相当于mov ecx,dword ptr ds:[esi]
;作用把存放在堆栈上的eax的值给ecx
004A1088    0FB69407 CB087A>movzx edx,byte ptr ds:[edi+eax-0x5085F735]
;相当于movzx edx,byte ptr ds:[edi]
;到这里eax参与解密dsp任务完成了，后面对eax的变形都是垃圾指令
004A1090    FEC0            inc al ;垃圾指令
004A1092    0f95c0          setne al ;垃圾指令
004A1095    66:c1c0 61      rol ax,0x61 ;垃圾指令
004A1099    32D3            xor dl,bl;开始解密edx，bl是密钥
004A109B    66:c1e8 a1      shr ax,0xa1 ;垃圾指令
004A109F    D0C2            rol dl,1
004A10A1    80C2 37         add dl,0x37
004A10A4    C1E0 1B         shl eax,0x1B ;垃圾指令
004A10A7    0F89 7C3C0500   jns 123_vmp.004F4D29

```

```
004F4D29    F6D2            not dl
004F4D2B    8D8440 A84E0DB1 lea eax,dword ptr ds:[eax+eax*2-0x4EF2B158]
 ;垃圾指令
004F4D32    FECA            dec dl
004F4D34    C1C8 14         ror eax,0x14 ;垃圾指令
004F4D37    c0f8 41         sar al,0x41 ;垃圾指令
004F4D3A    80F2 15         xor dl,0x15
004F4D3D    E8 10A6F8FF     call 123_vmp.0047F352

```

```
0047F352    FECA            dec dl
0047F354    8D04C5 1EEC0240 lea eax,dword ptr ds:[eax*8+0x4002EC1E]
;垃圾指令
0047F35B    58              pop eax
;平衡004F4D3D的call指令
0047F35C    32DA            xor bl,dl;bl生成新密钥
0047F35E    B8 85C0BF26     mov eax,0x26BFC085
0047F363    E8 0EAB0700     call 123_vmp.004F9E76

```

```
004F9E76    8D5414 04       lea edx,dword ptr ss:[esp+edx+0x4]
;这条指令的dsp = 4，这个dsp是用来平衡0047F363的call的
;这里是算出偏移为0x24虚拟寄存器的地址然后给edx
004F9E7A    C78404 7B3F40D9>mov dword ptr ss:[esp+eax-0x26BFC085],0x8AA262B0
;等效于mov dword ptr ss:[esp],0x8AA262B0
004F9E85    c18c44 f67e80b2>ror dword ptr ss:[eax*2+esp+0xb2807ef6],0x21
;等效于ror dword ptr ss:[esp],0x21
004F9E8D  ^ 0F80 2B14FFFF   jo 123_vmp.004EB2BE

```

```
004EB2BE    898C02 7B3F40D9 mov dword ptr ds:[edx+eax-0x26BFC085],ecx
;等效于 mov dword ptr ds:[edx],ecx
;这里的ecx在004A1081被赋值，偏移为0x24虚拟寄存器就是vm_eax
004EB2C5    8B8C47 F77E80B2 mov ecx,dword ptr ds:[edi+eax*2-0x4D7F8109]
;等效于 mov ecx,dword ptr ds:[edi+1]
;作用是从伪代码里面读取4字节到ecx
004EB2CC    33CB            xor ecx,ebx  ;开始解密ecx
004EB2CE    66:c1f8 42      sar ax,0x42
004EB2D2    0FC9            bswap ecx
004EB2D4    49              dec ecx
004EB2D5    8D90 15989EC4   lea edx,dword ptr ds:[eax-0x3B6167EB]
;等效于 mov edx,EB5E8836
004EB2DB    0FC9            bswap ecx
004EB2DD    c1e8 dc         shr eax,0xdc
004EB2E0    8154C4 F0 9C9EB>adc dword ptr ss:[esp+eax*8-0x10],0xD6B39E9C
;等效于 adc dword ptr ss:[esp],0xD6B39E9C
;[esp]的值来源于004F9E85的计算，此时esp = 0019FE7C
004EB2E8    194404 FE       sbb dword ptr ss:[esp+eax-0x2],eax
;等效于 sbb dword ptr ss:[esp],eax
004EB2EC    F7D9            neg ecx
004EB2EE    E8 0FBFF7FF     call 123_vmp.00467202

```

```
00467202    D1C9            ror ecx,1 ;ecx解密结束
00467204    52              push edx
;edx值来源 004EB2D5的计算
00467205    0FC11484        xadd dword ptr ss:[esp+eax*4],edx
;等效于 xadd dword ptr ss:[esp+8],edx
00467209    33D9            xor ebx,ecx ;生成新的密钥
0046720B    03E9            add ebp,ecx
;计算得出下一条handler地址
;这个ebp就是我们这条handler的开头004A107C
;算出下一条handler的地址004A107C
0046720D    E8 E6580800     call 123_vmp.004ECAF8

```

```
004ECAF8    8B5406 02       mov edx,dword ptr ds:[esi+eax+0x2]
;等效于 mov edx,dword ptr ds:[esi+4]
;从堆栈取出ebx的值给edx
004ECAFC    0FB6C8          movzx ecx,al
004ECAFF    FE4C84 05       dec byte ptr ss:[esp+eax*4+0x5]
;等效于dec byte ptr ss:[esp+0xD]
;执行完[0019FE7C] =  07635727 ,原来的值是07635827
004ECB03    8A4487 FD       mov al,byte ptr ds:[edi+eax*4-0x3]
;等效于 mov al,byte ptr ds:[edi+0x5]
;从伪代码取出一个字节到al
004ECB07    0FC9            bswap ecx
004ECB09    32C3            xor al,bl ;开始解密al
004ECB0B    c0c1 26         rol cl,0x26
004ECB0E    80C1 AE         add cl,0xAE
004ECB11    66:D3E9         shr cx,cl
004ECB14    D0C0            rol al,1
004ECB16    04 37           add al,0x37
004ECB18    33C9            xor ecx,ecx
;ecx = 0
004ECB1A    F6D0            not al
004ECB1C    894C8C 08       mov dword ptr ss:[esp+ecx*4+0x8],ecx
;等效于 mov dword ptr ss:[esp+0x8],ecx
004ECB20    0f97444c 0f     seta byte ptr ss:[ecx*2+esp+0xf]
;等效于 seta byte ptr ss:[esp+0xf]
;执行完[0019FE7C] =  00635727
004ECB25    FEC8            dec al
004ECB27    66:874C0C 06    xchg word ptr ss:[esp+ecx+0x6],cx
;等效于xchg word ptr ss:[esp+0x6],cx
;执行完[0019FE74] =  00008836 ，cx =  EB5E
004ECB2C    F69C0C AD14FFFF neg byte ptr ss:[esp+ecx-0xEB53]
;等效于 neg byte ptr ss:[esp+0xB]
004ECB33    34 15           xor al,0x15
004ECB35    66:c1f1 cb      sal cx,0xcb
004ECB39    C7844C 0020FEFF>mov dword ptr ss:[esp+ecx*2-0x1E000],0xD5936797
;等效于mov dword ptr ss:[esp],0xD5936797
004ECB44    C1AC0C 0C10FFFF>shr dword ptr ss:[esp+ecx-0xEFF4],0x3
;等效于shr dword ptr ss:[esp+0xC],0x3
;执行完[0019FE7C] =  000C6AE4
004ECB4C    FEC8            dec al
004ECB4E    66:FF844C 0920F>inc word ptr ss:[esp+ecx*2-0x1DFF7]
;等效于 inc word ptr ss:[esp+ 0x9]
;执行完[0019FE78] =    00000100
004ECB56    32D8            xor bl,al ;生成新的密钥
004ECB58    E8 ECE90000     call 123_vmp.004FB549

```

```
004FB549    8D4404 14       lea eax,dword ptr ss:[esp+eax+0x14]
;执行完  eax = 0019FE84
;这里其实就是计算虚拟寄存器的地址，偏移0x4
004FB54D    66:c18c0c 0810f>ror word ptr ss:[ecx+esp+0xffff1008],0xc4
;等效于 ror word ptr ss:[esp+0x8],0xc4
;执行完[0019FE74] =    00006883
004FB556    899488 0040FCFF mov dword ptr ds:[eax+ecx*4-0x3C000],edx
;edx的值来源于004ECAF8计算
;等效于 mov dword ptr ds:[eax+0],edx
;分析可以得出偏移0x4的虚拟寄存器就是vm_ebx
004FB55D    66:F7D1         not cx
004FB560    0f93840c 0df0ff>setae byte ptr ss:[ecx+esp-0xff3]
;等效于 setae byte ptr ss:[esp+0xc]
;执行完[0019FE78] =    00000101 
004FB568    8B840E 09F0FFFF mov eax,dword ptr ds:[esi+ecx-0xFF7]
;等效于mov eax,dword ptr ds:[esi+0x8]
;执行完 eax = 0019FF80
004FB56F    8DB431 0DF0FFFF lea esi,dword ptr ds:[ecx+esi-0xFF3]
;等效于 add esi，0xC
004FB576    81E1 B21EB0E6   and ecx,0xE6B01EB2
004FB57C    8D9409 3EBC8E95 lea edx,dword ptr ds:[ecx+ecx-0x6A7143C2]
;执行完  edx = 958ED9A2
004FB583    0FB68C4F A2E2FF>movzx ecx,byte ptr ds:[edi+ecx*2-0x1D5E]
;等效于 movzx ecx,byte ptr ds:[004277BA]
;从伪代码取出1个字节到ecx
004FB58B    83DF F9         sbb edi,-0x7
;这里才更新伪代码指针
004FB58E    025424 05       add dl,byte ptr ss:[esp+0x5]
;这里dl和前面计算得出的0x67相加
;通过下一条指令判断，这条是垃圾指令
004FB592    5A              pop edx
004FB593    81C2 90BDF9FF   add edx,0xFFF9BD90
;计算下一个代码块地址
;这里并不是计算handler地址，handler地址前面计算好了存在ebp
;结合后面分析这里是将handler的代码块拆成好几块并用jmp作连接
004FB599  - FFE2            jmp edx
;去执行下一个代码块 004888ED   

```

```
004888ED    8B5424 0C       mov edx,dword ptr ss:[esp+0xC]
004888F1    32CB            xor cl,bl       ;开始解密cl
004888F3    8D92 39BFB324   lea edx,dword ptr ds:[edx+0x24B3BF39]
;等效于 add edx，0x24B3BF39
004888F9    D0C1            rol cl,1
004888FB    80C1 37         add cl,0x37
004888FE    F6D1            not cl
00488900    E8 EB39FEFF     call 123_vmp.0046C2F0

```

```
0046C2F0    339414 EDD53FDB xor edx,dword ptr ss:[esp+edx-0x24C02A13]
;等效于 xor edx,dword ptr ss:[esp+0xA]
0046C2F7    FEC9            dec cl
0046C2F9    E8 5C4CFDFF     call 123_vmp.00440F5A

```

```
00440F5A    80F1 15         xor cl,0x15
00440F5D    875424 04       xchg dword ptr ss:[esp+0x4],edx
;执行完[0019FE6C ] =  25C12A1D  ，edx = 00488905
00440F61    81C2 81B7FBFF   add edx,0xFFFBB781 ;计算下一个代码块地址
00440F67  - FFE2            jmp edx  ;edx = 00444086

```

```
00444086    FEC9            dec cl ;解密cl结束
00444088    32D9            xor bl,cl ;生成新的密钥
0044408A    E8 87900A00     call 123_vmp.004ED116

```

```
004ED116    8B5424 04       mov edx,dword ptr ss:[esp+0x4]     
004ED11A    81C2 F9F80000   add edx,0xF8F9 ;计算下一个代码块地址
004ED120  ^ FFE2            jmp edx   ;edx =  0047BBF7       

```

```
0047BBF7    8D4C0C 1C       lea ecx,dword ptr ss:[esp+ecx+0x1C]
;执行完  ecx = 0019FEB0
;这里其实就是计算虚拟寄存器的地址，偏移0x30
0047BBFB    BA 0FB33B73     mov edx,0x733BB30F
0047BC00    875424 04       xchg dword ptr ss:[esp+0x4],edx
;执行完[0019FE68 ] =  733BB30F  ，edx = 0046C2FE
0047BC04    81C2 12EBFEFF   add edx,0xFFFEEB12 ;计算下一个代码块地址
0047BC0A  ^ FFE2            jmp edx  ;edx = 0045AE10

```

```
0045AE10    8901            mov dword ptr ds:[ecx],eax
;执行完[0019FEB0] =  0019FF80
;这里的eax在004FB568被赋值，偏移为0x30虚拟寄存器就是vm_ebp
0045AE12    58              pop eax  
0045AE13    05 2F400600     add eax,0x6402F;计算下一个代码块地址
0045AE18  - FFE0            jmp eax  ;eax = 004A80BE

```

```
004A80BE    B8 32580E4A     mov eax,0x4A0E5832
004A80C3    8D9400 364CB2CA lea edx,dword ptr ds:[eax+eax-0x354DB3CA]
;等效于 mov edx，0x5ECEFC9A
004A80CA    89AC04 CEA7F1B5 mov dword ptr ss:[esp+eax-0x4A0E5832],ebp
; ebp值来源于0046720B上的代码计算
;等效于 mov dword ptr ss:[esp],ebp
004A80D1    C2 1400         retn 0x14 ;这里才是真正去执行下一条handler

```

这条 handler 完成了 vm_eax(偏移是 0x24),Vm_ebx(偏移是 0x4),Vm_ebp(偏移是 0x30) 的填充，我们再看下一条 handler。

这条 handler 是我们上一条执行的 handler，完成了 vm_relocation(偏移是 0x28),Vm_esi(偏移是 0x2c),Vm_edx(偏移是 0x20) 的填充，我们再看下一条 handler。

这条 handler 是我们上一条执行的 handler，完成了 vm_edi(偏移是 0xC),Vm_eflags(偏移是 0x10),Vm_ecx(偏移是 0x40) 的填充， 这个 Vm_ecx 的偏移是 0x40，这和我们以前分析虚拟寄存器不一样，我们以前分析的虚拟寄存器总共 16 个，总的大小是 0x40，只能解释这个版本的虚拟寄存器大小和个数变得更大了。到这边就已经填充虚拟寄存器结束了，我们再看下一条 handler。

```
0043DC4E    8B0E            mov ecx,dword ptr ds:[esi]
; 这里是把00493A18执行call时压入堆栈的返回地址给ecx
0043DC50    BA 97778DEB     mov edx,0xEB8D7797 ;垃圾指令
0043DC55    0FB617          movzx edx,byte ptr ds:[edi]
; 从伪代码取出1个字节到edx
0043DC58    E8 09A20900     call 123_vmp.004D7E66

```

```
004D7E66    32D3            xor dl,bl  ;开始解密dl
004D7E68    B8 A1509D41     mov eax,0x419D50A1
004D7E6D    898404 5FAF62BE mov dword ptr ss:[esp+eax-0x419D50A1],eax
;等效于 mov dword ptr ss:[esp],eax
;执行完 [0019FE7C] = 0x419D50A1
004D7E74    D0C2            rol dl,1
004D7E76    FEC0            inc al
004D7E78    80C2 37         add dl,0x37
004D7E7B    FEC0            inc al
004D7E7D    c0e0 65         shl al,0x65
004D7E80    E8 FC8AF6FF     call 123_vmp.00440981

```

```
00440981    F6D2            not dl
00440983    66:F7D0         not ax
00440986    66:23C0         and ax,ax
00440989    FECA            dec dl
0044098B    32C0            xor al,al
0044098D    80F2 15         xor dl,0x15
00440990    24 BB           and al,0xBB
00440992    50              push eax
;执行完 [0019FE74] = 419DAF00
00440993    FECA            dec dl ;dl解密结束
00440995    66:c1ac04 01516>shr word ptr ss:[eax+esp+0xbe625101],0xa2
;等效于 shr word ptr ss:[esp+0x1],0xa2
;执行完 [0019FE74] = 41276B00
0044099E    c0c0 41         rol al,0x41
004409A1    32DA            xor bl,dl ;生成新的密钥
004409A3    25 A154BD47     and eax,0x47BD54A1
004409A8    8D5414 0C       lea edx,dword ptr ss:[esp+edx+0xC]
;等效于 lea edx,dword ptr ss:[esp+0x50]
004409AC    898C10 00FC62BE mov dword ptr ds:[eax+edx-0x419D0400],ecx
;等效于 mov dword ptr ds:[0019FEC4],ecx
;执行完 [0019FEc4] = 00493A1D
004409B3    8B8407 01FC62BE mov eax,dword ptr ds:[edi+eax-0x419D03FF]
;等效于 mov eax,dword ptr ds:[edi+1]
; 从伪代码取出4个字节到eax
004409BA    B9 81B084E7     mov ecx,0xE784B081
004409BF    874C24 04       xchg dword ptr ss:[esp+0x4],ecx
;执行完 [0019FEc4] = E784B081 ,ecx = 004D7E85
004409C3    81D1 1D87FDFF   adc ecx,0xFFFD871D ;计算下一个代码块地址
004409C9    FFE1            jmp ecx  ecx = 004B05A2

```

```
004B05A2    33C3            xor eax,ebx  ;开始解密eax
004B05A4    BA 95D43D18     mov edx,0x183DD495
004B05A9    FE8414 6E2BC2E7 inc byte ptr ss:[esp+edx-0x183DD492]
;等效于 inc byte ptr ss:[esp+0x3]
;执行完 [0019FE74] = 42276B00
004B05B0    0FC8            bswap eax
004B05B2    E8 B3DB0300     call 123_vmp.004EE16A

```

```
004EE16A    48              dec eax
004EE16B    0FBFCA          movsx ecx,dx
004EE16E    0FC8            bswap eax
004EE170    c1c1 fe         rol ecx,0xfe
004EE173    F7D8            neg eax
004EE175    FF840C E70A0080 inc dword ptr ss:[esp+ecx-0x7FFFF519]
;等效于 inc dword ptr ss:[esp+0xC]
;[esp+0xC]的值来源于 004D7E6D上代码的计算
004EE17C  ^ 0F84 0650F9FF   je 123_vmp.00483188 ;永远不会跳转
004EE182    D1C8            ror eax,1 ;eax解密结束
004EE184    F79C54 DE5684CF neg dword ptr ss:[esp+edx*2-0x307BA922]
;等效于  neg dword ptr ss:[esp+0x8]
;执行完 [0019FE78] = 187B4F7F
004EE18B    33D8            xor ebx,eax ;生成新的密钥
004EE18D    899454 E05684CF mov dword ptr ss:[esp+edx*2-0x307BA920],edx
;等效于  mov dword ptr ss:[esp+0xA],edx
;执行完 [0019FE78] = D4954F7F
;执行完 [0019FE7C] = 419D183D
004EE194    03E8            add ebp,eax
;这个ebp就是我们这条handler的开头0043DC4E
;算出下一条handler的地址00431F4E
004EE196    58              pop eax      
004EE197    05 3B5FFCFF     add eax,0xFFFC5F3B ;计算下一个代码块地址
004EE19C  - FFE0            jmp eax

```

```
004764F2    66:c18494 b4ad0>rol word ptr ss:[edx*4+esp+0x9f08adb4],0xa4
;等效于  rol word ptr ss:[esp+0x8],0xa4
;执行完 [0019FE7C] =    419D83D1
004764FB    0FB6C2          movzx eax,dl
004764FE    8B9416 6F2BC2E7 mov edx,dword ptr ds:[esi+edx-0x183DD491]
;等效于 mov edx,dword ptr ds:[esi+4]
00476505    66:FFC8         dec ax
00476508    F6D8            neg al
0047650A    66:298C0C E50A0>sub word ptr ss:[esp+ecx-0x7FFFF51B],cx
;等效于 sub word ptr ss:[esp+0xA],cx
;执行完 [0019FE7C] =  4C7883D1
00476512    8DB446 30FFFFFF lea esi,dword ptr ds:[esi+eax*2-0xD0]
;等效于 add esi,0x8
00476519    0FB68C39 E00A00>movzx ecx,byte ptr ds:[ecx+edi-0x7FFFF520]
;等效于  movzx ecx,byte ptr ds:[edi+0x5]
;从伪代码中读取一个字节到ecx
00476521    E8 96300100     call 123_vmp.004895BC

```

```
004895BC    c14c04 9b af    ror dword ptr ss:[eax+esp-0x65],0xaf
;等效于 ror dword ptr ss:[esp+0x7],0xaf
;执行完 [0019FE74] =  9E276B00
;执行完 [0019FE78] =  D4FE852A
004895C1    8DBC47 2EFFFFFF lea edi,dword ptr ds:[edi+eax*2-0xD2]
;等效于 add edi, 6
004895C8    8D0485 90EB8E45 lea eax,dword ptr ds:[eax*4+0x458EEB90]
;等效于 mov eax,458EED40
004895CF    32CB            xor cl,bl ;开始解密cl
004895D1    D0C1            rol cl,1
004895D3    C1E8 19         shr eax,0x19
004895D6    0F89 ABC00000   jns 123_vmp.00495687
;这里没去验证，猜测是永远跳转

```

```
00495687    58              pop eax    
--------中间代码省略--------
00495698    80F1 15         xor cl,0x15
0049569B    FEC9            dec cl    ;cl解密结束
0049569D    66:c17c24 04 e7 sar word ptr ss:[esp+0x4],0xe7
004956A3    32D9            xor bl,cl ;生成新的密钥
004956A5    8B4424 03       mov eax,dword ptr ss:[esp+0x3]
004956A9  ^ E9 B434FCFF     jmp 123_vmp.00458B62

```

```
00458B62    58              pop eax
00458B63    8D4C0C 0C       lea ecx,dword ptr ss:[esp+ecx+0xC]
;等效于 lea ecx,dword ptr ss:[esp+0xC]
00458B67    66:c1c8 4e      ror ax,0x4e
00458B6B    288404 F4635DAF sub byte ptr ss:[esp+eax-0x50A29C0C],al
;等效于 sub byte ptr ss:[esp+1],al
00458B72    118404 F7635DAF adc dword ptr ss:[esp+eax-0x50A29C09],eax
;等效于 adc dword ptr ss:[esp+4],eax
00458B79    899401 F3635DAF mov dword ptr ds:[ecx+eax-0x50A29C0D],edx
;等效于      mov dword ptr ds:[19fe80],edx
00458B80    818C04 FB635DAF>or dword ptr ss:[esp+eax-0x50A29C05],0x3E0DD6BA
;等效于      or dword ptr ss:[esp+8],0x3E0DD6BA
00458B8B    c1a404 fb635daf>shl dword ptr ss:[eax+esp+0xaf5d63fb],0x39
;等效于      shl dword ptr ss:[esp+0x8],0x39
00458B93    89AC04 F3635DAF mov dword ptr ss:[esp+eax-0x50A29C0D],ebp
;等效于      mov dword ptr ss:[esp],ebp
00458B9A    C2 0800         retn 0x8 ;去执行下一条handler

```

上面这条 handler 执行下来，做了两件事将 0x33e43063 写入偏移为 0 的虚拟寄存器中和将 0x493a1d 写入偏移为 0x44 的虚拟寄存器中，这里我把写入的两个值修改了也不影响运行，所以这条 handler，应该就是干扰我们分析的，前面做那么多注释浪费我大量的时间，这里基本上用的一些手段都明白了，后面我就标记一些关键的部分了，主要看伪代码读取数据后解密完成之后做了什么。我们看下一条 handler。

```
00431F4E    68 8BBD8323     push 0x2383BD8B
00431F53    8B0F            mov ecx,dword ptr ds:[edi]
;从伪代码读取4字节到ecx
00431F55    33CB            xor ecx,ebx  ;开始解密ecx
00431F57    D1C1            rol ecx,1
--------中间代码省略--------
00431F75    F7D8            neg eax
00431F77    41              inc ecx  ;ecx解密结束
00431F78    E9 5B880100     jmp 123_vmp.0044A7D8

```

```
0044A7D8    66:F79414 8DBDF>not word ptr ss:[esp+edx-0x4273]
0044A7E0    c1f0 29         sal eax,0x29
0044A7E3    33D9            xor ebx,ecx  ;生成密钥
0044A7E5    66:878414 8CBDF>xchg word ptr ss:[esp+edx-0x4274],ax
0044A7ED    03E9            add ebp,ecx  ;算出下一条handler地址 00470423
0044A7EF    8BC8            mov ecx,eax
0044A7F1    66:C1C0 0B      rol ax,0xB
0044A7F5    8B8C57 1A7BFFFF mov ecx,dword ptr ds:[edi+edx*2-0x84E6]
;从伪代码读取4字节到ecx
0044A7FC    8D9402 BE1D3607 lea edx,dword ptr ds:[edx+eax+0x7361DBE]
0044A803    F7DA            neg edx
0044A805    33CB            xor ecx,ebx ;开始解密ecx
0044A807    c1c8 9b         ror eax,0x9b
0044A80A    F7D9            neg ecx
0044A80C    FE8C04 A17C7DEF dec byte ptr ss:[esp+eax-0x1082835F]
0044A813    0FC9            bswap ecx
0044A815    81B404 A17C7DEF>xor dword ptr ss:[esp+eax-0x1082835F],0x853B7190
0044A820    0F81 9E2F0A00   jno 123_vmp.004ED7C4 ;  永远都跳

```

```
004ED7C4    66:818C04 A27C7>or word ptr ss:[esp+eax-0x1082835E],0x9CB0
004ED7CE    C1C9 03         ror ecx,0x3 ;ecx解密结束
004ED7D1    8D8C08 AAB031F9 lea ecx,dword ptr ds:[eax+ecx-0x6CE4F56]
;解密出00415035，这个地址就是我们加密第二条代码
;jmp后面的地址
004ED7D8    66:0FC1C2       xadd dx,ax
004ED7DC    089484 36D1F5BD or byte ptr ss:[esp+eax*4-0x420A2ECA],dl
004ED7E3    33D9            xor ebx,ecx ;生成密钥
004ED7E5    E8 40520000     call 123_vmp.004F2A2A

```

```
004F2A2A    898C06 49747DEF mov dword ptr ds:[esi+eax-0x10828BB7],ecx
;存放00415035在0019FF70
004F2A31    23C0            and eax,eax
004F2A33    8D0455 0B38AA82 lea eax,dword ptr ds:[edx*2-0x7D55C7F5]
004F2A3A    50              push eax
004F2A3B    0FB68C3A F6F0BA>movzx ecx,byte ptr ds:[edx+edi-0x450F0A]
;从伪代码读取1字节到ecx
004F2A43    8DBC57 E5E175FF lea edi,dword ptr ds:[edi+edx*2-0x8A1E1B]
004F2A4A    c1ea c1         shr edx,0xc1
004F2A4D    66:118414 7878D>adc word ptr ss:[esp+edx-0x228788],ax
004F2A55    32CB            xor cl,bl  ;开始解密cl
004F2A57    80C1 A3         add cl,0xA3
--------中间代码省略--------
004F2A78    D0C9            ror cl,1
004F2A7A    80C1 20         add cl,0x20 ;cl解密结束
004F2A7D    F75424 08       not dword ptr ss:[esp+0x8]
004F2A81    895424 04       mov dword ptr ss:[esp+0x4],edx 
004F2A85    32D9            xor bl,cl ;生成密钥
004F2A87    0FCA            bswap edx 
004F2A89    c1ca f1         ror edx,0xf1
004F2A8C    FECA            dec dl
004F2A8E    8D4C0C 0C       lea ecx,dword ptr ss:[esp+ecx+0xC]
;获得vm_ecx的地址
004F2A92    4A              dec edx  
004F2A93    c1f0 7a         sal eax,0x7a
004F2A96    66:c1f8 89      sar ax,0x89
004F2A9A    8B8411 229018A0 mov eax,dword ptr ds:[ecx+edx-0x5FE76FDE]
;取出vm_ecx的值
004F2AA1    898416 1A9018A0 mov dword ptr ds:[esi+edx-0x5FE76FE6],eax
;将前面取出的值存放在0019FF6C
004F2AA8    8DB416 1A9018A0 lea esi,dword ptr ds:[esi+edx-0x5FE76FE6]
;等效于 sub esi，8
004F2AAF    59              pop ecx  
004F2AB0    5A              pop edx    
004F2AB1    59              pop ecx
;执行上面三条指令后esp = 0019FE80，也就是虚拟寄存器基地址
004F2AB2  ^ 0F84 A31BF6FF   je 123_vmp.0045465B ;永远跳转

```

```
0045465B    B9 975DA731     mov ecx,0x31A75D97
00454660    E8 8CA40300     call 123_vmp.0048EAF1

```

```
0048EAF1    59              pop ecx  
0048EAF2    8D8C24 88000000 lea ecx,dword ptr ss:[esp+0x88]
;相当于 add esp，0x88 加上 mov ecx，esp
0048EAF9    E9 1D1E0600     jmp 123_vmp.004F091B

```

```
004F091B  ^\E9 A25BFDFF     jmp 123_vmp.004C64C2

```

```
004C64C2    3BF1            cmp esi,ecx
; 这条比较指令中，ecx 永远比 esi 大
004C64C4    E8 447DFBFF     call 123_vmp.0047E20D

```

```
0047E20D    68 88E2049F     push 0x9F04E288
0047E212    8D6424 08       lea esp,dword ptr ss:[esp+0x8]
0047E216    0F87 1CAF0300   ja 123_vmp.004B9138 ;永远跳转

```

```
004B9138  ^\E9 F029FDFF     jmp 123_vmp.0048BB2D

```

```
0048BB2D  ^\FFE5            jmp ebp  ;执行下一条handler

```

这条 handler 做了两件事，将解密出来的 00415035 在 0019FF70，将 vm_ecx 取出来存放在 0019FF6C。我们接着看下一条 handler。

```
00470423    B8 2FCA3DC6     mov eax,0xC63DCA2F
00470428    0FBFC8          movsx ecx,ax
0047042B    8B07            mov eax,dword ptr ds:[edi] ;读取伪代码
0047042D    33C3            xor eax,ebx ;开始解密
0047042F    F7D8            neg eax
00470431    E8 C4FB0500     call 123_vmp.004CFFFA

```

```
004CFFFA    0BC9            or ecx,ecx
--------中间代码省略--------
004D0008    59              pop ecx    
004D0009    05 0934B409     add eax,0x9B43409 ;解密完成
;这里解密出来的就是第一个加密的指令的数值 12345678
004D000E    33D8            xor ebx,eax ;生成密钥
004D0010    68 0E1FBE9C     push 0x9CBE1F0E
004D0015    8BD0            mov edx,eax
004D0017  ^ E9 2FFCF8FF     jmp 123_vmp.0045FC4B

```

```
0045FC4B    B9 012F314D     mov ecx,0x4D312F01
0045FC50    51              push ecx   
0045FC51    0FB6840F 03D1CE>movzx eax,byte ptr ds:[edi+ecx-0x4D312EFD]
0045FC59    32C3            xor al,bl ;开始解密
0045FC5B    59              pop ecx   
--------中间代码省略--------
0045FC6F    66:c1f1 6f      sal cx,0x6f
0045FC73    0F86 D3220300   jbe 123_vmp.00491F4C ;永远跳转

```

```
00491F4C    FEC8            dec al
00491F4E    59              pop ecx
00491F4F    F7D1            not ecx
00491F51    806424 03 2C    and byte ptr ss:[esp+0x3],0x2C
00491F56    34 15           xor al,0x15
00491F58    FEC8            dec al             ;解密完成
00491F5A    66:c16424 01 cf shl word ptr ss:[esp+0x1],0xcf
00491F60    c10c24 3e       ror dword ptr ss:[esp],0x3e
00491F64    32D8            xor bl,al            ;生成密钥
00491F66    D30424          rol dword ptr ss:[esp],cl
00491F69    8D4404 04       lea eax,dword ptr ss:[esp+eax+0x4]
;算出偏移为0的虚拟寄存器的地址
00491F6D    0FC9            bswap ecx
00491F6F    D3C9            ror ecx,cl
00491F71    899408 4C4053CC mov dword ptr ds:[eax+ecx-0x33ACBFB4],edx
;将值12345678存入偏移为0的虚拟寄存器
00491F78    c0b40c 4e4053cc>sal byte ptr ss:[ecx+esp+0xcc53404e],0x24
00491F80    878C4C 9880A698 xchg dword ptr ss:[esp+ecx*2-0x67597F68],ecx
00491F87    8B840F F7FFFFF3 mov eax,dword ptr ds:[edi+ecx-0xC000009]
00491F8E    41              inc ecx
00491F8F    33C3            xor eax,ebx  ;开始解密
00491F91    E9 36B10000     jmp 123_vmp.0049D0CC

```

```
0049D0CC    D3E9            shr ecx,cl
--------中间代码省略--------
0049D0EA    0f9ac2          setpe dl
0049D0ED    D1C8            ror eax,1   ;解密完成
0049D0EF    C1FA 03         sar edx,0x3
0049D0F2    8D0CD5 B0FD1529 lea ecx,dword ptr ds:[edx*8+0x2915FDB0]
0049D0F9    33D8            xor ebx,eax  ;生成密钥
0049D0FB    03E8            add ebp,eax
;计算下一条handler地址
0049D0FD    0FB6C2          movzx eax,dl
0049D100    0FB69438 29FFFF>movzx edx,byte ptr ds:[eax+edi-0xD7]
0049D108    8DBC39 5A03EAD6 lea edi,dword ptr ds:[ecx+edi-0x2915FCA6]
0049D10F    02CD            add cl,ch
0049D111    0FC9            bswap ecx
0049D113    32D3            xor dl,bl  ;开始解密
0049D115    c0e0 86         shl al,0x86
--------中间代码省略--------
0049D12D    0FC0E9          xadd cl,ch
0049D130    80D2 20         adc dl,0x20  ;解密完成
0049D133    32DA            xor bl,dl   ;生成密钥
0049D135  ^ E9 FC2FFFFF     jmp 123_vmp.00490136

```

```
00490136    0FC8            bswap eax
00490138    03D4            add edx,esp
;算出vm_eflags的地址
0049013A    8B8402 ADFCFFFF mov eax,dword ptr ds:[edx+eax-0x353]
;取出vm_eflags的值
00490141    8946 FC         mov dword ptr ds:[esi-0x4],eax
;将取出的值存放在19ff68
00490144    E8 F935FAFF     call 123_vmp.00433742

```

```
00433742    66:c1f9 69      sar cx,0x69
00433746    83D6 FC         adc esi,-0x4
00433749    59              pop ecx     
0043374A    E9 0C0F0200     jmp 123_vmp.0045465B

```

```
0045465B    B9 975DA731     mov ecx,0x31A75D97
00454660    E8 8CA40300     call 123_vmp.0048EAF1

```

```
0048EAF1    59              pop ecx       
0048EAF2    8D8C24 88000000 lea ecx,dword ptr ss:[esp+0x88]
0048EAF9    E9 1D1E0600     jmp 123_vmp.004F091B

```

```
004F091B  ^\E9 A25BFDFF     jmp 123_vmp.004C64C2

```

```
004C64C2    3BF1            cmp esi,ecx
004C64C4    E8 447DFBFF     call 123_vmp.0047E20D

```

```
0047E20D    68 88E2049F     push 0x9F04E288
0047E212    8D6424 08       lea esp,dword ptr ss:[esp+0x8]
0047E216    0F87 1CAF0300   ja 123_vmp.004B9138 ;永远跳转

```

```
004B9138  ^\E9 F029FDFF     jmp 123_vmp.0048BB2D

```

```
0048BB2D   /FFE5            jmp ebp  ;执行下一条handler

```

这条 handler 做了两件事，将解密出来的 12345678 存入偏移为 0 的虚拟寄存器，将 vm_eflags 取出来存放在 19ff68。我们接着看下一条 handler。

```
0048BBD9    B9 8CAA9922     mov ecx,0x2299AA8C
0048BBDE    0FB68C39 745566>movzx ecx,byte ptr ds:[ecx+edi-0x2299AA8C]
0048BBE6    32CB            xor cl,bl ;开始解密
0048BBE8    B8 256D2E96     mov eax,0x962E6D25
0048BBED    80C1 A3         add cl,0xA3
0048BBF0    E8 17640500     call 123_vmp.004E200C

```

```
004E200C    D0C9            ror cl,1
004E200E    80C1 2C         add cl,0x2C
004E2011    D0C9            ror cl,1
004E2013    80C1 20         add cl,0x20  ;解密完成
004E2016    66:FFC8         dec ax
004E2019    890424          mov dword ptr ss:[esp],eax
004E201C    FE4424 01       inc byte ptr ss:[esp+0x1]
004E2020    32D9            xor bl,cl    ;生成密钥
004E2022    FF0C24          dec dword ptr ss:[esp] 
004E2025    E8 E4C4FFFF     call 123_vmp.004DE50E

```

```
004DE50E    0FB7D0          movzx edx,ax
004DE511    8D4C0C 08       lea ecx,dword ptr ss:[esp+ecx+0x8]
;算出vm_edi的地址
004DE515    8B840A DC92FFFF mov eax,dword ptr ds:[edx+ecx-0x6D24]
;取出vm_edi的值
004DE51C    c08c94 774bfeff>ror byte ptr ss:[edx*4+esp+0xfffe4b77],0x61
004DE524    52              push edx
004DE525    319454 B825FFFF xor dword ptr ss:[esp+edx*2-0xDA48],edx
004DE52C    898416 D892FFFF mov dword ptr ds:[esi+edx-0x6D28],eax
;将vm_edi的值存放在19ff64
004DE533    E8 AA92FBFF     call 123_vmp.004977E2

```

```
004977E2    899454 C025FFFF mov dword ptr ss:[esp+edx*2-0xDA40],edx
004977E9    899414 DC92FFFF mov dword ptr ss:[esp+edx-0x6D24],edx
004977F0    8B8417 DD92FFFF mov eax,dword ptr ds:[edi+edx-0x6D23]
004977F7    F79414 E892FFFF not dword ptr ss:[esp+edx-0x6D18]
004977FE    59              pop ecx    
004977FF    33C3            xor eax,ebx  ;开始解密
00497801    51              push ecx
--------中间代码省略--------
00497818    D3E9            shr ecx,cl
0049781A    F7D0            not eax     ;解密完成
0049781C    33D8            xor ebx,eax ;生成密钥
0049781E    0ACA            or cl,dl
00497820    66:FFCA         dec dx
00497823    13E8            adc ebp,eax ;计算下一条handler地址
00497825    0FB6843A E292FF>movzx eax,byte ptr ds:[edx+edi-0x6D1E]
0049782D    51              push ecx
0049782E    8DBC3A E392FFFF lea edi,dword ptr ds:[edx+edi-0x6D1D]
00497835    E9 D2F90100     jmp 123_vmp.004B720C

```

```
004B720C    32C3            xor al,bl ;开始解密
004B720E    66:C1E9 05      shr cx,0x5
--------中间代码省略--------
004B722C    66:23CA         and cx,dx
004B722F    14 20           adc al,0x20  ;解密完成
004B7231    F7D1            not ecx
004B7233    32D8            xor bl,al    ;生成密钥
004B7235    c17c14 03 6d    sar dword ptr ss:[edx+esp+0x3],0x6d
004B723A    8D4404 10       lea eax,dword ptr ss:[esp+eax+0x10]
;算出vm_edx的地址
004B723E    0314D0          add edx,dword ptr ds:[eax+edx*8]
;取出vm_edx的值
004B7241    8956 F8         mov dword ptr ds:[esi-0x8],edx
;将vm_edx的值存放在19ff60
004B7244    BA B8A0A013     mov edx,0x13A0A0B8
--------中间代码省略--------
004B7258  ^ E9 FED3F9FF     jmp 123_vmp.0045465B

```

```
0045465B    B9 975DA731     mov ecx,0x31A75D97
00454660    E8 8CA40300     call 123_vmp.0048EAF1

```

```
0048EAF1    59              pop ecx        
0048EAF2    8D8C24 88000000 lea ecx,dword ptr ss:[esp+0x88]
0048EAF9    E9 1D1E0600     jmp 123_vmp.004F091B

```

```
004F091B  ^\E9 A25BFDFF     jmp 123_vmp.004C64C2

```

```
004C64C2    3BF1            cmp esi,ecx
004C64C4    E8 447DFBFF     call 123_vmp.0047E20D

```

```
0047E20D    68 88E2049F     push 0x9F04E288
0047E212    8D6424 08       lea esp,dword ptr ss:[esp+0x8]
0047E216    0F87 1CAF0300   ja 123_vmp.004B9138  ;永远跳转

```

```
004B9138  ^\E9 F029FDFF     jmp 123_vmp.0048BB2D

```

```
0048BB2D   /FFE5            jmp ebp     ;执行下一条handler

```

这条 handler 做了两件事，将 vm_edi 的值存放在 19ff64，将 vm_edx 的值存放在 19ff60。我们接着看下一条 handler。

```
0048BBD9    B9 8CAA9922     mov ecx,0x2299AA8C
0048BBDE    0FB68C39 745566>movzx ecx,byte ptr ds:[ecx+edi-0x2299AA8C]
0048BBE6    32CB            xor cl,bl   ;开始解密
0048BBE8    B8 256D2E96     mov eax,0x962E6D25
0048BBED    80C1 A3         add cl,0xA3
0048BBF0    E8 17640500     call 123_vmp.004E200C

```

```
004E200C    D0C9            ror cl,1
004E200E    80C1 2C         add cl,0x2C
004E2011    D0C9            ror cl,1
004E2013    80C1 20         add cl,0x20   ;解密完成
004E2016    66:FFC8         dec ax
004E2019    890424          mov dword ptr ss:[esp],eax
004E201C    FE4424 01       inc byte ptr ss:[esp+0x1]
004E2020    32D9            xor bl,cl       ;生成密钥
004E2022    FF0C24          dec dword ptr ss:[esp]  
004E2025    E8 E4C4FFFF     call 123_vmp.004DE50E

```

```
004DE50E    0FB7D0          movzx edx,ax
004DE511    8D4C0C 08       lea ecx,dword ptr ss:[esp+ecx+0x8]
;算出vm_esi的地址
004DE515    8B840A DC92FFFF mov eax,dword ptr ds:[edx+ecx-0x6D24]
;取出vm_esi的值
004DE51C    c08c94 774bfeff>ror byte ptr ss:[edx*4+esp+0xfffe4b77],0x61
004DE524    52              push edx
004DE525    319454 B825FFFF xor dword ptr ss:[esp+edx*2-0xDA48],edx
004DE52C    898416 D892FFFF mov dword ptr ds:[esi+edx-0x6D28],eax
;将vm_esi的值存放在19ff5C
004DE533    E8 AA92FBFF     call 123_vmp.004977E2

```

```
004977E2    899454 C025FFFF mov dword ptr ss:[esp+edx*2-0xDA40],edx
004977E9    899414 DC92FFFF mov dword ptr ss:[esp+edx-0x6D24],edx
004977F0    8B8417 DD92FFFF mov eax,dword ptr ds:[edi+edx-0x6D23]
004977F7    F79414 E892FFFF not dword ptr ss:[esp+edx-0x6D18]
004977FE    59              pop ecx       
004977FF    33C3            xor eax,ebx  ;开始解密
00497801    51              push ecx
--------中间代码省略--------
00497818    D3E9            shr ecx,cl
0049781A    F7D0            not eax        ;解密完成
0049781C    33D8            xor ebx,eax   ;生成密钥
0049781E    0ACA            or cl,dl
00497820    66:FFCA         dec dx
00497823    13E8            adc ebp,eax ;计算下一条handler地址
00497825    0FB6843A E292FF>movzx eax,byte ptr ds:[edx+edi-0x6D1E]
0049782D    51              push ecx
0049782E    8DBC3A E392FFFF lea edi,dword ptr ds:[edx+edi-0x6D1D]
00497835    E9 D2F90100     jmp 123_vmp.004B720C

```

```
004B720C    32C3            xor al,bl  ;开始解密
004B720E    66:C1E9 05      shr cx,0x5
--------中间代码省略--------
004B722C    66:23CA         and cx,dx
004B722F    14 20           adc al,0x20   ;解密完成
004B7231    F7D1            not ecx
004B7233    32D8            xor bl,al      ;生成密钥
004B7235    c17c14 03 6d    sar dword ptr ss:[edx+esp+0x3],0x6d
004B723A    8D4404 10       lea eax,dword ptr ss:[esp+eax+0x10]
;算出vm_ebp的地址
004B723E    0314D0          add edx,dword ptr ds:[eax+edx*8]
;取出vm_ebp的值
004B7241    8956 F8         mov dword ptr ds:[esi-0x8],edx
;将vm_ebp的值存放在19ff58
004B7244    BA B8A0A013     mov edx,0x13A0A0B8
004B7249    0FC9            bswap ecx
--------中间代码省略--------
004B7258  ^ E9 FED3F9FF     jmp 123_vmp.0045465B

```

```
0045465B    B9 975DA731     mov ecx,0x31A75D97
00454660    E8 8CA40300     call 123_vmp.0048EAF1

```

```
0048EAF1    59              pop ecx
0048EAF2    8D8C24 88000000 lea ecx,dword ptr ss:[esp+0x88]
0048EAF9    E9 1D1E0600     jmp 123_vmp.004F091B

```

```
004F091B  ^\E9 A25BFDFF     jmp 123_vmp.004C64C2

```

```
004C64C2    3BF1            cmp esi,ecx
004C64C4    E8 447DFBFF     call 123_vmp.0047E20D

```

```
0047E20D    68 88E2049F     push 0x9F04E288
0047E212    8D6424 08       lea esp,dword ptr ss:[esp+0x8]
0047E216    0F87 1CAF0300   ja 123_vmp.004B9138   ;永远跳转

```

```
004B9138  ^\E9 F029FDFF     jmp 123_vmp.0048BB2D

```

```
0048BB2D   /FFE5            jmp ebp    ;执行下一条handler

```

这条 handler 做了两件事，将将 vm_esi 的值存放在 19ff5C，将将 vm_ebp 的值存放在 19ff58 。我们接着看下一条 handler。

```
0048BBD9    B9 8CAA9922     mov ecx,0x2299AA8C
0048BBDE    0FB68C39 745566>movzx ecx,byte ptr ds:[ecx+edi-0x2299AA8C]
0048BBE6    32CB            xor cl,bl ;开始解密
0048BBE8    B8 256D2E96     mov eax,0x962E6D25
0048BBED    80C1 A3         add cl,0xA3
0048BBF0    E8 17640500     call 123_vmp.004E200C

```

```
004E200C    D0C9            ror cl,1
004E200E    80C1 2C         add cl,0x2C
004E2011    D0C9            ror cl,1
004E2013    80C1 20         add cl,0x20   ;解密完成
004E2016    66:FFC8         dec ax
004E2019    890424          mov dword ptr ss:[esp],eax
004E201C    FE4424 01       inc byte ptr ss:[esp+0x1]
004E2020    32D9            xor bl,cl    ;生成密钥
004E2022    FF0C24          dec dword ptr ss:[esp]  
004E2025    E8 E4C4FFFF     call 123_vmp.004DE50E

```

```
004DE50E    0FB7D0          movzx edx,ax
004DE511    8D4C0C 08       lea ecx,dword ptr ss:[esp+ecx+0x8]
;算出vm_ebx的地址
004DE515    8B840A DC92FFFF mov eax,dword ptr ds:[edx+ecx-0x6D24]
;取出vm_ebx的值
004DE51C    c08c94 774bfeff>ror byte ptr ss:[edx*4+esp+0xfffe4b77],0x61
004DE524    52              push edx
004DE525    319454 B825FFFF xor dword ptr ss:[esp+edx*2-0xDA48],edx
004DE52C    898416 D892FFFF mov dword ptr ds:[esi+edx-0x6D28],eax
;将vm_ebx的值存放在19ff54
004DE533    E8 AA92FBFF     call 123_vmp.004977E2

```

```
004977E2    899454 C025FFFF mov dword ptr ss:[esp+edx*2-0xDA40],edx
004977E9    899414 DC92FFFF mov dword ptr ss:[esp+edx-0x6D24],edx
004977F0    8B8417 DD92FFFF mov eax,dword ptr ds:[edi+edx-0x6D23]
004977F7    F79414 E892FFFF not dword ptr ss:[esp+edx-0x6D18]
004977FE    59              pop ecx                
004977FF    33C3            xor eax,ebx ;开始解密
00497801    51              push ecx
00497802    66:0FC18C94 7A4>xadd word ptr ss:[esp+edx*4-0x1B486],cx
0049780B    D1C8            ror eax,1
0049780D    0FC18CD4 EC96FC>xadd dword ptr ss:[esp+edx*8-0x36914],ecx
00497815    0FC8            bswap eax
00497817    48              dec eax
00497818    D3E9            shr ecx,cl
0049781A    F7D0            not eax   ;解密完成
0049781C    33D8            xor ebx,eax     ;生成密钥
0049781E    0ACA            or cl,dl
00497820    66:FFCA         dec dx
00497823    13E8            adc ebp,eax ;计算下一条handler地址
00497825    0FB6843A E292FF>movzx eax,byte ptr ds:[edx+edi-0x6D1E]
0049782D    51              push ecx
0049782E    8DBC3A E392FFFF lea edi,dword ptr ds:[edx+edi-0x6D1D]
00497835    E9 D2F90100     jmp 123_vmp.004B720C

```

```
004B720C    32C3            xor al,bl ;开始解密
004B720E    66:C1E9 05      shr cx,0x5
--------中间代码省略--------
004B722C    66:23CA         and cx,dx
004B722F    14 20           adc al,0x20   ;解密完成
004B7231    F7D1            not ecx
004B7233    32D8            xor bl,al     ;生成密钥
004B7235    c17c14 03 6d    sar dword ptr ss:[edx+esp+0x3],0x6d
004B723A    8D4404 10       lea eax,dword ptr ss:[esp+eax+0x10]
;算出偏移为0的虚拟寄存器的地址
004B723E    0314D0          add edx,dword ptr ds:[eax+edx*8]
;取出偏移为0的虚拟寄存器的值 12345678
004B7241    8956 F8         mov dword ptr ds:[esi-0x8],edx
;将 12345678  存放在19ff50
004B7244    BA B8A0A013     mov edx,0x13A0A0B8
--------中间代码省略--------
004B7258  ^ E9 FED3F9FF     jmp 123_vmp.0045465B

```

```
0045465B    B9 975DA731     mov ecx,0x31A75D97
00454660    E8 8CA40300     call 123_vmp.0048EAF1

```

```
0048EAF1    59              pop ecx  
0048EAF2    8D8C24 88000000 lea ecx,dword ptr ss:[esp+0x88]
0048EAF9    E9 1D1E0600     jmp 123_vmp.004F091B

```

```
004F091B  ^\E9 A25BFDFF     jmp 123_vmp.004C64C2

```

```
004C64C2    3BF1            cmp esi,ecx
004C64C4    E8 447DFBFF     call 123_vmp.0047E20D

```

```
0047E20D    68 88E2049F     push 0x9F04E288
0047E212    8D6424 08       lea esp,dword ptr ss:[esp+0x8]
0047E216    0F87 1CAF0300   ja 123_vmp.004B9138  ;永远跳转

```

```
004B9138  ^\E9 F029FDFF     jmp 123_vmp.0048BB2D

```

```
0048BB2D   /FFE5            jmp ebp  ;执行下一条handler

```

这条 handler 做了两件事，将将 vm_ebx 的值存放在 19ff54，将 12345678 存放在 19ff50。我们接着看下一条 handler。

```
0049B90B  ^\E9 16F0FBFF     jmp 123_vmp.0045A926

```

```
0045A926    BF 0235A29B     mov edi,0x9BA23502 ;垃圾指令
0045A92B    8BE6            mov esp,esi        ;修改esp为esi的值，为恢复寄存器做准备
0045A92D    0FBFEF          movsx ebp,di        ;垃圾指令
0045A930    58              pop eax       
;这条指令执行完，eax = 12345678。这就模拟了第一条加密的指令
0045A931    5B              pop ebx            ;恢复ebx
0045A932    8BCF            mov ecx,edi      ;垃圾指令
0045A934    5D              pop ebp                ;恢复ebp  
0045A935    0FB7D1          movzx edx,cx      ;垃圾指令
0045A938    5E              pop esi          ;恢复esi
0045A939    81EA 29E99930   sub edx,0x3099E929  ;垃圾指令
0045A93F    5A              pop edx             ;恢复edx
0045A940    66:F7D9         neg cx             ;垃圾指令
0045A943    5F              pop edi                ;恢复edi  
0045A944    F7D9            neg ecx             ;垃圾指令
0045A946    33C9            xor ecx,ecx         ;清零，参与后面的解密dsp
0045A948    9D              popfd                ;恢复eflags
0045A949    E8 B6810500     call 123_vmp.004B2B04

```

```
004B2B04    890C0C          mov dword ptr ss:[esp+ecx],ecx ;垃圾指令
004B2B07    8B4C0C 04       mov ecx,dword ptr ss:[esp+ecx+0x4]
; 等效为 mov ecx,dword ptr ss:[esp+0x4]
;[esp+0x4] 的值在执行004F2A9A上的代码赋予的
;恢复ecx 
004B2B0B    68 0AED3613     push 0x1336ED0A
004B2B10    E8 D3BCFAFF     call 123_vmp.0045E7E8

```

```
0045E7E8    8D6424 10       lea esp,dword ptr ss:[esp+0x10]
0045E7EC    C3              retn
;离开虚拟机
;同时也是模拟 第二条我们加密的指令 jmp 00415035

```

到此为止全部代码分析完成了，感觉和分析 3.0.0 版本还是费劲不少，主要就是 dsp 加密，搞得代码看起来很不舒服，但是也是有规律可循，分析久了，那些代码可以跳过，哪些代码需要留意，也能总结出来，熟能生巧应该嘛。

[[培训] 内核驱动高级班，冲击 BAT 一流互联网大厂工作，每周日 13:00-18:00 直播授课](https://www.kanxue.com/book-section_list-173.htm)

最后于 5 天前 被阿强编辑 ，原因：

[#调试逆向](forum-4-1-1.htm) [#VM 保护](forum-4-1-4.htm)

上传的附件：

*   [vmp.rar](javascript:void(0);) （830.76kb，17 次下载）