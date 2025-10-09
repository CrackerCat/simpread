> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-288681.htm)

> [原创]VmProtect.3.9.4 分析之虚拟机流程

终于到了可以开始分析 VmProtect.3.9.4 了，像极了玩游戏来到了最终 boss 环节，我们玩过游戏的都知道，打 boss 是一件不容易的是请，因为作为整个游戏里最强大的生物，他的血量防御极其高，但是打赢了通关之后还是很有成就感的。要做到能够完全分析他还有很长的一段路要走，从 1.6 到这个版本只是分析他的一个分支 [虚拟化]，千里之行始于足下，我们没有办法做到一口气，把全部知识点都消化了，但是我们可以像愚公移山一样，所谓量变引起质变，就是这个意思。

这次给大家的附件依旧没有虚拟机主程序，然后我说下，设置啥的，首先加密的是入口点后面的的下面两行代码，和上篇帖子一样

```
0040A5A8 >  B8 AAAAAAAA     mov eax,0xAAAAAAAA
0040A5AD    BB BBBBBBBB     mov ebx,0xBBBBBBBB
0040A5B2    B9 CCCCCCCC     mov ecx,0xCCCCCCCC
0040A5B7    BA DDDDDDDD     mov edx,0xDDDDDDDD
0040A5BC    BE 51515151     mov esi,0x51515151
0040A5C1    BF D1D1D1D1     mov edi,0xD1D1D1D1
0040A5C6    B8 78563412     mov eax,0x12345678 
0040A5CB    E9 65AA0000     jmp 123.00415035
0040A5D0    C3              retn
0040A5D1    90              nop

```

加密的是`0040A5C6 B8 78563412 mov eax,0x12345678`和`0040A5CB E9 65AA0000 jmp 123.00415035`这两条代码，然后选项那边，把是都改成否，英文版的，yes 全改 no，其他的都不要动。我们把编译成功的 123.vmp.exe 拖到 OD 上

```
0040A5C6  |.  57            push edi
0040A5C7  |.  53            push ebx 
0040A5C8  |.  E8 B4AB0B00   call 123_vmp.004C5181
0040A5CD  |.  59            pop ecx        
0040A5CE  \.- FFE5          jmp ebp

```

我们可以看到这里又变了，我们之前分析的版本都是开头改成 jmp 指令，我们单步走往下看，

```
004C5181    9C              pushfd 
004C5182    BF 9C9C823B     mov edi,0x3B829C9C
004C5187    BB 8C0F063D     mov ebx,0x3D060F8C
004C518C    81F3 A36618C2   xor ebx,0xC21866A3
;执行完eflags值变化了
004C5192    8BBC3C 70637DC4 mov edi,dword ptr ss:[esp+edi-0x3B829C90]
;可简化成 mov edi,dword ptr ss:[esp+0xC]
;这里也使用了我一篇帖子讲的dsp加密,这里dsp = 0xc
;执行完edi值又恢复成D1D1D1D1
004C5199    C74424 0C 0895A>mov dword ptr ss:[esp+0xC],0x8BA49508
;执行完 0019FF70存放8BA49508
004C51A1    8B5C24 08       mov ebx,dword ptr ss:[esp+0x8]
;执行完edi值又恢复成BBBBBBBB
004C51A5    E8 3294FFFF     call 123_vmp.004BE5DC

```

```
004BE5DC    FF7424 04       push dword ptr ss:[esp+0x4]
;执行完 0019FF5C存放00000246
004BE5E0    9D              popfd
;执行完  eflags值恢复成 0246
004BE5E1    8D6424 10       lea esp,dword ptr ss:[esp+0x10]
;可简化成 add esp,10
004BE5E5    E8 4C7F0300     call 123_vmp.004F6536

```

```
004F6536  ^\E9 6C6DFEFF     jmp 123_vmp.004DD2A7

```

```
004DD2A7    52              push edx  ;edx入栈
004DD2A8    E8 72250100     call 123_vmp.004EF81F

```

```
004EF81F    896C24 00       mov dword ptr ss:[esp],ebp ;ebp入栈
004EF823  ^ E9 273CFAFF     jmp 123_vmp.0049344F

```

```
0049344F    E8 85530000     call 123_vmp.004987D9

```

```
004987D9    897424 00       mov dword ptr ss:[esp],esi  ;esi入栈
004987DD    9C              pushfd ;eflags入栈
004987DE    894C24 10       mov dword ptr ss:[esp+0x10],ecx ;ecx入栈
004987E2    BD A9203EC7     mov ebp,0xC73E20A9
004987E7    0FBFD5          movsx edx,bp
004987EA    0FBECA          movsx ecx,dl
004987ED    BD 00000000     mov ebp,0x0
004987F2  ^ E9 AF47FFFF     jmp 123_vmp.0048CFA6

```

```
0048CFA6    53              push ebx   ;ebx入栈
0048CFA7    8BF2            mov esi,edx
0048CFA9  ^ E9 169EFFFF     jmp 123_vmp.00486DC4

```

```
00486DC4    55              push ebp ;relocation(重定位寄存器)入栈
00486DC5    F7D1            not ecx
00486DC7    0FBEDA          movsx ebx,dl
00486DCA    57              push edi  ;edi入栈
00486DCB    8BB454 CEBEFFFF mov esi,dword ptr ss:[esp+edx*2-0x4132]
;可简化成 mov esi,dword ptr ss:[esp+0x20]
00486DD2    8DB41E 5FA00588 lea esi,dword ptr ds:[esi+ebx-0x77FA5FA1]
;执行完 esi = 13AA3510
00486DD9    81F6 1ABD17EC   xor esi,0xEC17BD1A
00486DDF    F7D6            not esi
00486DE1    0FA3C9          bt ecx,ecx
00486DE4    66:2BD3         sub dx,bx
00486DE7    0FBEFB          movsx edi,bl
00486DEA    46              inc esi
00486DEB    66:23FB         and di,bx
00486DEE    0F8C 53360600   jl 123_vmp.004EA447   ;永远跳转

```

```
004EA447    13F5            adc esi,ebp
;esi重定位，这里重定位值ebp = 0
004EA449    898454 20BEFFFF mov dword ptr ss:[esp+edx*2-0x41E0],eax
;eax入栈
004EA450    8BFC            mov edi,esp
004EA452    0FB7C2          movzx eax,dx
004EA455    c1e1 b2         shl ecx,0xb2
004EA458  ^ 0F82 E54AFAFF   jb 123_vmp.0048EF43  ;永远不跳
004EA45E    8D9C57 3CBDFFFF lea ebx,dword ptr ds:[edi+edx*2-0x42C4]
;指令可以拆分为，sub edi，C4 和 mov ebx，edi 执行完ebx = 0019FE8C
004EA465    66:0FBAE9 2E    bts cx,0x2E
004EA46A    51              push ecx
004EA46B    8BE3            mov esp,ebx
; ebx 赋值给 esp ，esp = 0019FE8C
004EA46D  ^ E9 6834F9FF     jmp 123_vmp.0047D8DA

```

```
0047D8DA    BD 3FCD892B     mov ebp,0x2B89CD3F
0047D8DF    8BDE            mov ebx,esi
;将伪代码地址给ebx，ebx存放密钥
0047D8E1    B9 00000000     mov ecx,0x0
0047D8E6    8D94AD 1AB329E2 lea edx,dword ptr ss:[ebp+ebp*4-0x1DD64CE6]
;执行完 edx = BBDAB555
0047D8ED    0FBFC5          movsx eax,bp
0047D8F0    2BD9            sub ebx,ecx ;垃圾指令
0047D8F2    BD BEB1BAF8     mov ebp,0xF8BAB1BE
0047D8F7    0FB7CD          movzx ecx,bp
0047D8FA    8D2D F2D84700   lea ebp,dword ptr ds:[0x47D8F2]
;将代码中的某个地址0047D8F2存放到ebp，参与解密下一条handler
0047D900    22C9            and cl,cl
0047D902    0FB7C1          movzx eax,cx
0047D905    8DB486 0439FDFF lea esi,dword ptr ds:[esi+eax*4-0x2C6FC]
;可简化成 sub esi，4
0047D90C    66:98           cbw
0047D90E    8B940E 424EFFFF mov edx,dword ptr ds:[esi+ecx-0xB1BE]
;可简化成 mov edx,dword ptr ds:[esi]
;从伪代码读取4字节到edx
0047D915    0FBAE0 97       bt eax,0x97
0047D919    33D3            xor edx,ebx
;开始解密handler地址相加数
;值得注意的是这里的密钥ebx的值和上一篇帖子分析3.8.1有一点区别
;首次解密密钥ebx = esi初始值，但是这里的密钥ebx = esi初始值 + 4
;结合后面的分析伪代码是倒着存放的，esi的初始值应该是 当前esi + 4
0047D91B    33C9            xor ecx,ecx
0047D91D    86C8            xchg al,cl
0047D91F    c1c1 cc         rol ecx,0xcc
0047D922    F7DA            neg edx
0047D924    86CD            xchg ch,cl
0047D926  ^ E9 DC29FFFF     jmp 123_vmp.00470307

```

```
00470307    42              inc edx
00470308    0FB3C0          btr eax,eax
0047030B    F7D2            not edx
0047030D    49              dec ecx
0047030E    E8 655B0700     call 123_vmp.004E5E78

```

```
004E5E78    03C9            add ecx,ecx
004E5E7A    F7DA            neg edx
004E5E7C    0FCA            bswap edx
004E5E7E    c1e8 d1         shr eax,0xd1
004E5E81    F7DA            neg edx        ;解密handler地址相加数完成
004E5E83    F6D8            neg al
004E5E85    0FBAE9 08       bts ecx,0x8
004E5E89    33DA            xor ebx,edx   ;生成新的密钥
004E5E8B    C7840C 42FEE9FF>mov dword ptr ss:[esp+ecx-0x1601BE],0x540F46B2
004E5E96    03EA            add ebp,edx   ;算出下一条handler地址 0050AA79
004E5E98    99              cdq
004E5E99    58              pop eax     
004E5E9A    50              push eax
004E5E9B    89AC04 4EB9F0AB mov dword ptr ss:[esp+eax-0x540F46B2],ebp  
004E5EA2    C3              retn          ;执行下一条handler

```

执行到这里就完成了寄存器入栈和其他初始化操作了，现在我们可以大致分析出来，ebx 作用是存放密钥，esp 是存放虚拟寄存器基地址，ebp 用于存放 handler 的地址，esi 存放伪代码的指针，edi 的作用是参与堆栈的读写操作，类似执行真实指令时候的 esp。下面就是寄存器入栈后在堆栈上的分布表

<table><thead><tr><th>堆栈地址</th><th>寄存器值</th><th>寄存器名称</th></tr></thead><tbody><tr><td>0019FF50</td><td>D1D1D1D1</td><td>edi</td></tr><tr><td>0019FF54</td><td>00000000</td><td>relocation</td></tr><tr><td>0019FF58</td><td>BBBBBBBB</td><td>ebx</td></tr><tr><td>0019FF5C</td><td>00000246</td><td>eflags</td></tr><tr><td>0019FF60</td><td>51515151</td><td>esi</td></tr><tr><td>0019FF64</td><td>0019FF80</td><td>ebp</td></tr><tr><td>0019FF68</td><td>DDDDDDDD</td><td>edx</td></tr><tr><td>0019FF6C</td><td>CCCCCCCC</td><td>ecx</td></tr><tr><td>0019FF70</td><td>AAAAAAAA</td><td>eax</td></tr></tbody></table>

我们来看第一条 handler

```
0050AA79    8B4E FC         mov ecx,dword ptr ds:[esi-0x4]
;这里从 esi - 4 取出4字节给ecx，结合前面0047D919上代码执行时
;ebx的值，我们可以推断伪代码存放顺序是倒着的
0050AA7C    B8 879910BE     mov eax,0xBE109987
0050AA81    0FB6D0          movzx edx,al
0050AA84    33CB            xor ecx,ebx  ;开始解密handler地址相加数
0050AA86    92              xchg eax,edx
0050AA87    F7D1            not ecx
0050AA89    0FB3C2          btr edx,eax
0050AA8C    D1C1            rol ecx,1
0050AA8E    c0e2 e6         shl dl,0xe6
0050AA91    66:c1e2 8c      shl dx,0x8c
0050AA95    c0e0 e3         shl al,0xe3
0050AA98    49              dec ecx
0050AA99    0AC0            or al,al
0050AA9B    F7D9            neg ecx      ;解密handler地址相加数完成
0050AA9D    33D9            xor ebx,ecx  ;生成新的密钥
0050AA9F    03E9            add ebp,ecx  ;算出下一条handler地址 0050AA79
0050AAA1    66:0FA3D2       bt dx,dx
0050AAA5    8B5407 C8       mov edx,dword ptr ds:[edi+eax-0x38]
;从堆栈中取出edi的值
0050AAA9    c1c8 77         ror eax,0x77
0050AAAC    66:13C0         adc ax,ax
0050AAAF    0FB68C06 FB1FFF>movzx ecx,byte ptr ds:[esi+eax-0xE005]
;从伪代码读取1字节到ecx
0050AAB7    32CB            xor cl,bl ;开始解密虚拟寄存器偏移
0050AAB9    98              cwde
0050AABA    D0C1            rol cl,1
0050AABC    F6D9            neg cl
0050AABE    25 14AD1DED     and eax,0xED1DAD14
0050AAC3    80D1 9A         adc cl,0x9A
0050AAC6    D0C9            ror cl,1
0050AAC8    23C0            and eax,eax
0050AACA    80F1 87         xor cl,0x87
0050AACD    FEC9            dec cl     ;解密虚拟寄存器偏移完成
0050AACF    c1f8 2b         sar eax,0x2b
0050AAD2    32D9            xor bl,cl ;生成新的密钥
0050AAD4    0AC4            or al,ah
0050AAD6    c1e0 8b         shl eax,0x8b
0050AAD9    03CC            add ecx,esp
;算出偏移为0x4虚拟寄存器地址
0050AADB    66:0BC0         or ax,ax
0050AADE    891421          mov dword ptr ds:[ecx],edx
;将edi的值存放到偏移为0x4虚拟寄存器中
;这个虚拟寄存器我们叫做vm_edi
0050AAE1    0FBBC0          btc eax,eax
0050AAE4    50              push eax
0050AAE5    c17c24 00 5f    sar dword ptr ss:[esp],0x5f
0050AAEA    8B4427 04       mov eax,dword ptr ds:[edi+0x4]
;从堆栈中取出relocation(重定位值 0)的值到eax
0050AAEE    0FB64E FA       movzx ecx,byte ptr ds:[esi-0x6]
;从伪代码读取1字节到ecx
0050AAF2    32CB            xor cl,bl  ;开始解密虚拟寄存器偏移
0050AAF4    E8 53A1FCFF     call 123_vmp.004D4C4C

```

```
004D4C4C    80C1 3B         add cl,0x3B
004D4C4F    FF4424 04       inc dword ptr ss:[esp+0x4]
004D4C53    D0C1            rol cl,1
004D4C55    66:c14c24 06 6a ror word ptr ss:[esp+0x6],0x6a
004D4C5B    68 178C28B7     push 0xB7288C17
004D4C60    8B5424 00       mov edx,dword ptr ss:[esp]   
004D4C64    80D9 B3         sbb cl,0xB3
004D4C67    D0C1            rol cl,1
004D4C69    875424 04       xchg dword ptr ss:[esp+0x4],edx
;取出上一个执行的call压入的返回地址参与算出下一个代码块的地址
004D4C6D    81C2 3B5CFFFF   add edx,0xFFFF5C3B
004D4C73  - FFE2            jmp edx
;这里在我分析3.0.0讲过这里不是去执行handler
;这个jmp指令起到连接代码作用

```

```
00500734  ^\E9 2E75FBFF     jmp 123_vmp.004B7C67

```

```
004B7C67    FEC1            inc cl     ;解密虚拟寄存器偏移完成
004B7C69    8B5424 08       mov edx,dword ptr ss:[esp+0x8]
004B7C6D    32D9            xor bl,cl  ;生成新的密钥
004B7C6F    FF4414 01       inc dword ptr ss:[esp+edx+0x1]
004B7C73    E8 0B26FBFF     call 123_vmp.0046A283

```

```
0046A283    2AD2            sub dl,dl
0046A285    8D4C0C 10       lea ecx,dword ptr ss:[esp+ecx+0x10]
;虽然说esp是虚拟寄存器基地址，但是这里经常执行call指令
;导致esp值变化所以这里的dsp值10也就是平衡前面的堆栈变化
;esp + 0x10 = 0019FE8C,这个值也就是虚拟寄存器基地址
;算出偏移为0x2c虚拟寄存器地址
0046A289    4A              dec edx
0046A28A    135424 06       adc edx,dword ptr ss:[esp+0x6]
0046A28E    1BD2            sbb edx,edx
0046A290    890421          mov dword ptr ds:[ecx],eax
;将relocation的值存放到偏移为0x2c虚拟寄存器中
;这个虚拟寄存器我们叫做vm_relocation
0046A293    0FB64424 0C     movzx eax,byte ptr ss:[esp+0xC]
0046A298    8B5447 08       mov edx,dword ptr ds:[edi+eax*2+0x8]
;从堆栈中取出ebx的值
0046A29C    0F964484 09     setbe byte ptr ss:[esp+eax*4+0x9]
0046A2A1    8D8C00 A6DB0B28 lea ecx,dword ptr ds:[eax+eax+0x280BDBA6]
0046A2A8    8D7C38 0C       lea edi,dword ptr ds:[eax+edi+0xC]
0046A2AC    0FC0C8          xadd al,cl
0046A2AF    898404 5AFFFFFF mov dword ptr ss:[esp+eax-0xA6],eax
0046A2B6    8A8430 53FFFFFF mov al,byte ptr ds:[eax+esi-0xAD]
;从伪代码读取1字节到al
0046A2BD    c1f9 45         sar ecx,0x45
0046A2C0    1B8C8C A984FEFA sbb ecx,dword ptr ss:[esp+ecx*4-0x5017B57]
0046A2C7    59              pop ecx    
0046A2C8    8DB44E ADFEFFFF lea esi,dword ptr ds:[esi+ecx*2-0x153]
;更新esi的值
0046A2CF    c0bc8c 6bfdffff>sar byte ptr ss:[ecx*4+esp-0x295],0x23
0046A2D7    32CD            xor cl,ch
0046A2D9    32C3            xor al,bl ;开始解密虚拟寄存器偏移
0046A2DB    66:D3C9         ror cx,cl
0046A2DE    0f93c1          setae cl
0046A2E1    0f93840c 0568ff>setae byte ptr ss:[ecx+esp+0xffff6805]
0046A2E9    1AC1            sbb al,cl
0046A2EB    66:0FB3C9       btr cx,cx
0046A2EF    49              dec ecx
0046A2F0    c0c1 67         rol cl,0x67
0046A2F3    F6D8            neg al
0046A2F5    2AC1            sub al,cl
0046A2F7    F6D8            neg al   ;解密虚拟寄存器偏移完成
0046A2F9    FEC9            dec cl
0046A2FB    FEC9            dec cl
0046A2FD    51              push ecx
0046A2FE    32D8            xor bl,al   ;生成新的密钥
0046A300    8D4404 10       lea eax,dword ptr ss:[esp+eax+0x10]
;算出偏移为0x38虚拟寄存器地址
0046A304    D3C1            rol ecx,cl
0046A306    D34C24 00       ror dword ptr ss:[esp],cl
0046A30A    0FC04C24 08     xadd byte ptr ss:[esp+0x8],cl
0046A30F    891420          mov dword ptr ds:[eax],edx
;将ebx的值存放到偏移为0x38虚拟寄存器中，这个虚拟寄存器我们叫做vm_ebx
0046A312    81D9 23EAA807   sbb ecx,0x7A8EA23
0046A318    8D9489 BBCB8F2A lea edx,dword ptr ds:[ecx+ecx*4+0x2A8FCBBB]
0046A31F    0F82 A5690300   jb 123_vmp.004A0CCA    ;永远不跳
0046A325    89AC54 0CD978B7 mov dword ptr ss:[esp+edx*2-0x488726F4],ebp 
0046A32C    C2 0C00         retn 0xC ;执行下一条handler

```

这条 handler 填充了三个虚拟寄存器，然后第二条要执行的 handler 地址还是 0050AA79，这边我就不再分析了，记录一下填充的虚拟寄存器就行了，分别是，将 eflags 的值存放到偏移为 0x8 虚拟寄存器中，将 esi 的值存放到偏移为 0x1c 虚拟寄存器中，将 ebp 的值存放到偏移为 0x40 虚拟寄存器中, 接着第三条要执行的 handler 地址还是 0050AA79，再记录一下填充的虚拟寄存器，分别是，将 edx 的值存放到偏移为 0x3c 虚拟寄存器中，将 ecx 的值存放到偏移为 0x20 虚拟寄存器中，将 eax 的值存放到偏移为 0xC 虚拟寄存器中。到这里所有虚拟寄存器全部填充完毕。这里我制作一个表格记录一下虚拟寄存器的分布图

<table><thead><tr><th>堆栈地址</th><th>虚拟寄存器</th><th>虚拟寄存器</th><th>虚拟寄存器</th><th>虚拟寄存器</th></tr></thead><tbody><tr><td>19fe8c</td><td>未占用</td><td>edi</td><td>eflags</td><td>eax</td></tr><tr><td>19fe9c</td><td>未占用</td><td>未占用</td><td>未占用</td><td>esi</td></tr><tr><td>19feac</td><td>ecx</td><td>未占用</td><td>未占用</td><td>vm_relocation</td></tr><tr><td>19febc</td><td>未占用</td><td>未占用</td><td>ebx</td><td>edx</td></tr><tr><td>19fecc</td><td>ebp</td><td>未占用</td><td>未占用</td><td>未占用</td></tr></tbody></table>

第四要执行的 handler 地址是 004A5D91, 我们接着往下看

```
004A5D91    8B4E FC         mov ecx,dword ptr ds:[esi-0x4]
;从伪代码读取4字节到ecx
004A5D94    B8 A97FAEBA     mov eax,0xBAAE7FA9
004A5D99    0FB7D0          movzx edx,ax
004A5D9C    0FC0E0          xadd al,ah
004A5D9F    33CB            xor ecx,ebx
;开始解密我们vm的第一条指令中immediate，也就是12345678
004A5DA1    E9 CB560000     jmp 123_vmp.004AB471

```

```
004AB471    c0fa c2         sar dl,0xc2
004AB474    C1C9 02         ror ecx,0x2
004AB477    FEC8            dec al
004AB479    0AC2            or al,dl
004AB47B    0FC9            bswap ecx
004AB47D    41              inc ecx
004AB47E    D1C1            rol ecx,1 ;解密immediate完成
004AB480    2AC4            sub al,ah
004AB482    33D9            xor ebx,ecx  ;生成新的密钥
004AB484    898C17 1280FFFF mov dword ptr ds:[edi+edx-0x7FEE],ecx
;将immediate存放在 0019FF70
004AB48B    E8 DD38FFFF     call 123_vmp.0049ED6D

```

```
0049ED6D    58              pop eax     
0049ED6E    05 364FFBFF     add eax,0xFFFB4F36
0049ED73  - FFE0            jmp eax ;下一个代码块地址为 004603C6

```

```
004603C6    8B8C16 0E80FFFF mov ecx,dword ptr ds:[esi+edx-0x7FF2]
;从伪代码读取4字节到ecx
004603CD    0FB6C2          movzx eax,dl
004603D0    0FABC2          bts edx,eax  
004603D3    13C0            adc eax,eax  
004603D5    33CB            xor ecx,ebx ;开始解密handler地址相加数
004603D7    66:c1ea e8      shr dx,0xe8
004603DB    8D84C2 005E9AFA lea eax,dword ptr ds:[edx+eax*8-0x565A200]
004603E2    F7D1            not ecx
004603E4    1BD0            sbb edx,eax  
004603E6    c1c8 d0         ror eax,0xd0
004603E9    E8 BAED0A00     call 123_vmp.0050F1A8

```

```
0050F1A8    D1C1            rol ecx,1
0050F1AA    58              pop eax        
0050F1AB    05 4C5D0000     add eax,0x5D4C
0050F1B0  - FFE0            jmp eax ;下一个代码块地址为 0046613A

```

```
0046613A    8D9492 3F98159E lea edx,dword ptr ds:[edx+edx*4-0x61EA67C1]
00466141    c1c2 db         rol edx,0xdb
00466144    49              dec ecx
00466145    8BC2            mov eax,edx
00466147    F7D9            neg ecx      ;解密handler地址相加数完成
00466149    66:33D2         xor dx,dx
0046614C    0FBAFA 16       btc edx,0x16
00466150    33D9            xor ebx,ecx  ;生成新的密钥
00466152    80EA 8B         sub dl,0x8B
00466155    03E9            add ebp,ecx  ;算出下一条handler地址 004D24DD
00466157    E8 D5B30000     call 123_vmp.00471531

```

```
00471531    5A              pop edx                   
00471532    81C2 3693FEFF   add edx,0xFFFE9336
00471538  - FFE2            jmp edx ;下一个代码块地址为 0044F492

```

```
0044F492    0FBFC8          movsx ecx,ax
0044F495    8B4C27 FC       mov ecx,dword ptr ds:[edi-0x4]
;取出immediate存放在ecx
0044F499    0FB64426 F7     movzx eax,byte ptr ds:[esi-0x9]
;从伪代码读取1字节到eax
0044F49E    BA 3C25BCAF     mov edx,0xAFBC253C
0044F4A3    F6D2            not dl
0044F4A5    66:c1f2 8f      sal dx,0x8f
0044F4A9    83D6 F6         adc esi,-0xA ;更新esi的值
0044F4AC    E9 C3060800     jmp 123_vmp.004CFB74

```

```
004CFB74    32C3            xor al,bl ;开始解密虚拟寄存器偏移
004CFB76    FEC0            inc al
004CFB78    52              push edx
004CFB79    66:0B5424 01    or dx,word ptr ss:[esp+0x1]
004CFB7E    5A              pop edx
004CFB7F    D0C0            rol al,1
004CFB81    32D6            xor dl,dh
004CFB83    FEC0            inc al
004CFB85    FECA            dec dl
004CFB87    23D2            and edx,edx
004CFB89    c1ea 42         shr edx,0x42
004CFB8C    F6D8            neg al
004CFB8E    0FBAE2 9C       bt edx,0x9C
004CFB92    1C 37           sbb al,0x37
004CFB94    0FA3D2          bt edx,edx
004CFB97    F6D8            neg al
004CFB99    8D94D2 2221380C lea edx,dword ptr ds:[edx+edx*8+0xC382122]
004CFBA0    c0f2 42         sal dl,0x42
004CFBA3    D0C0            rol al,1
004CFBA5    66:F7DA         neg dx
004CFBA8    F6DA            neg dl
004CFBAA    FEC8            dec al
004CFBAC    81F2 155AB82D   xor edx,0x2DB85A15
004CFBB2    4A              dec edx
004CFBB3    F6D8            neg al
004CFBB5    0AD6            or dl,dh
004CFBB7    F6D2            not dl
004CFBB9    c0c2 27         rol dl,0x27
004CFBBC    D0C8            ror al,1
004CFBBE    66:c1ca 69      ror dx,0x69
004CFBC2    FEC0            inc al
004CFBC4    8D9412 919A9CE3 lea edx,dword ptr ds:[edx+edx-0x1C63656F]
004CFBCB    34 B8           xor al,0xB8   ;解密虚拟寄存器偏移完成
004CFBCD    E8 6EF5FEFF     call 123_vmp.004BF140

```

```
004BF140    c1f2 3c         sal edx,0x3c
004BF143    32D8            xor bl,al  ;生成新的密钥
004BF145    0FB3D2          btr edx,edx
004BF148    8D4404 04       lea eax,dword ptr ss:[esp+eax+0x4]
;算出偏移为0x10虚拟寄存器地址
004BF14C    C78414 00000090>mov dword ptr ss:[esp+edx-0x70000000],0xBEAA0B07
004BF157    8D1455 B4FD3F97 lea edx,dword ptr ds:[edx*2-0x68C0024C]
004BF15E    898C10 4C02C088 mov dword ptr ds:[eax+edx-0x773FFDB4],ecx
;将immediate的值存放到偏移为0x10虚拟寄存器中，这个虚拟寄存器参与还原我们vm的第一条指令
004BF165    0FB7CA          movzx ecx,dx
004BF168    89AC14 4C02C088 mov dword ptr ss:[esp+edx-0x773FFDB4],ebp 
004BF16F    C3              retn   ;执行下一条handler 004D24DD

```

上面这条 handler 就干了一件事，将我们 vm 的第一条指令的 immediate 给存放到虚拟寄存器中，我们继续看第五条 handler

```
004D24DD    B9 BD5E15C1     mov ecx,0xC1155EBD
004D24E2    E8 F73CF9FF     call 123_vmp.004661DE

```

```
004661DE    8D048D A43E85C5 lea eax,dword ptr ds:[ecx*4-0x3A7AC15C]
004661E5    81EE 04000000   sub esi,0x4 ;更新esi的值
004661EB    98              cwde
004661EC    BA 98679C71     mov edx,0x719C6798
004661F1    C78414 6898638E>mov dword ptr ss:[esp+edx-0x719C6798],0x840A4532
004661FC    8B8416 6898638E mov eax,dword ptr ds:[esi+edx-0x719C6798]
;从伪代码读取4字节到eax
00466203    49              dec ecx
00466204    33C3            xor eax,ebx
;开始解密第二条vm的指令的displacement部分也就jmp后面的地址00415035
00466206    C1C8 02         ror eax,0x2
00466209    66:899414 68986>mov word ptr ss:[esp+edx-0x719C6798],dx
00466211    59              pop ecx      
00466212    66:0FA3D2       bt dx,dx
00466216    0FC8            bswap eax
00466218    81F1 343F0D02   xor ecx,0x20D3F34
0046621E    0F93C1          setnb cl
00466221    40              inc eax
00466222    D3F1            sal ecx,cl
00466224    D1C0            rol eax,1 ;解密displacement完成
00466226    33D8            xor ebx,eax  ;生成新的密钥
00466228    8DBC39 FA4FF1F3 lea edi,dword ptr ds:[ecx+edi-0xC0EB006]
0046622F    8984CF F07F8A9F mov dword ptr ds:[edi+ecx*8-0x60758010],eax
;将displacement 存放在 19ff70
00466236    8DB44E F89FE2E7 lea esi,dword ptr ds:[esi+ecx*2-0x181D6008]
;更新esi的值
0046623D    D3F2            sal edx,cl
0046623F    8B8C4E FC9FE2E7 mov ecx,dword ptr ds:[esi+ecx*2-0x181D6004]
;从伪代码读取4字节到ecx
00466246    8D8492 913921BA lea eax,dword ptr ds:[edx+edx*4-0x45DEC66F]
0046624D    66:c1f8 4f      sar ax,0x4f
00466251    33CB            xor ecx,ebx ;开始解密handler地址相加数
00466253    C1C1 03         rol ecx,0x3
00466256    F7D2            not edx               
00466258    66:0FBBC2       btc dx,ax
0046625C    49              dec ecx
0046625D    E9 A4330A00     jmp 123_vmp.00509606

```

```
00509606    4A              dec edx
00509607    66:0FABC2       bts dx,ax
0050960B    F7D9            neg ecx
0050960D    48              dec eax
0050960E    66:FFC8         dec ax
00509611    66:2BD0         sub dx,ax
00509614    81F1 2D3A3FBD   xor ecx,0xBD3F3A2D ;解密handler地址相加数完成
0050961A    66:2BC2         sub ax,dx
0050961D    15 81860595     adc eax,0x95058681
00509622    15 2CAF9EA2     adc eax,0xA29EAF2C
00509627    33D9            xor ebx,ecx  ;生成新的密钥
00509629    8D8490 AC2A1838 lea eax,dword ptr ds:[eax+edx*4+0x38182AAC]
00509630    03E9            add ebp,ecx ;算出下一条handler地址004D84C6
00509632  ^ E9 0C45FFFF     jmp 123_vmp.004FDB43

```

```
004FDB43    B8 05E6A7EC     mov eax,0xECA7E605
004FDB48    8D8424 84000000 lea eax,dword ptr ss:[esp+0x84]
004FDB4F    E8 BFD4FEFF     call 123_vmp.004EB013

```

```
004EB013  ^\E9 8525F7FF     jmp 123_vmp.0045D59D

```

```
0045D59D    3BF8            cmp edi,eax
;edi永远比eax大
0045D59F    E8 0F12FFFF     call 123_vmp.0044E7B3

```

```
0044E7B3  ^\E9 4963FEFF     jmp 123_vmp.00434B01

```

```
00434B01    E8 7B510D00     call 123_vmp.00509C81

```

```
00509C81    8D6424 0C       lea esp,dword ptr ss:[esp+0xC]
00509C85    0F87 68000000   ja 123_vmp.00509CF3    ;永远跳转

```

```
00509CF3   /E9 A9290000     jmp 123_vmp.0050C6A1

```

```
0050C6A1  ^\FFE5            jmp ebp     ;执行下一条handler 004D84C6 

```

上面这条 handler 就做了一件事，将 displacement 存放在 19ff70。我们再看第六条 handler

```
004D84C6    B9 B5AF82AB     mov ecx,0xAB82AFB5
004D84CB    8BD1            mov edx,ecx
004D84CD    81EE 04000000   sub esi,0x4 ;更新esi的值
004D84D3    8B0426          mov eax,dword ptr ds:[esi]
;从伪代码读取4字节到eax
004D84D6    33C3            xor eax,ebx ;开始解密handler地址相加数
004D84D8    c1c2 62         rol edx,0x62
004D84DB    D1C0            rol eax,1
004D84DD    86D1            xchg cl,dl
004D84DF    F7D0            not eax
004D84E1    0FABD2          bts edx,edx
004D84E4    1D A6069A43     sbb eax,0x439A06A6
004D84E9    0FC8            bswap eax  ;解密handler地址相加数完成
004D84EB    66:87CA         xchg dx,cx
004D84EE    42              inc edx
004D84EF    33D8            xor ebx,eax  ;生成新的密钥
004D84F1    51              push ecx
004D84F2    03E8            add ebp,eax ;算出下一条handler地址004D55D9
004D84F4    51              push ecx
004D84F5    F75C24 04       neg dword ptr ss:[esp+0x4]
004D84F9    66:0FC14C24 01  xadd word ptr ss:[esp+0x1],cx
004D84FF    81DE 00000000   sbb esi,0x0
;更新esi的值
;等效于 sub esi,0x1
004D8505    86CE            xchg dh,cl
004D8507    0FB60426        movzx eax,byte ptr ds:[esi]
;从伪代码读取1字节到eax
004D850B  ^ E9 74AEF6FF     jmp 123_vmp.00443384

```

```
00443384    32C3            xor al,bl ;开始解密虚拟寄存器偏移
00443386    34 2B           xor al,0x2B
00443388    33CA            xor ecx,edx
0044338A  ^ 0F81 DCF3FEFF   jno 123_vmp.0043276C   ;永远跳转

```

```
0043276C    E8 FB940A00     call 123_vmp.004DBC6C

```

```
004DBC6C    F6D0            not al
004DBC6E    1C 3E           sbb al,0x3E
004DBC70    F6D0            not al
004DBC72    FEC0            inc al   ;解密虚拟寄存器偏移完成
004DBC74    09940C 8CC357FA or dword ptr ss:[esp+ecx-0x5A83C74],edx
004DBC7B    32D8            xor bl,al  ;生成新的密钥
004DBC7D    66:0BC9         or cx,cx
004DBC80    8D4404 0C       lea eax,dword ptr ss:[esp+eax+0xC]
;算出偏移为0x10虚拟寄存器地址
004DBC84    8B8C48 1087AFF4 mov ecx,dword ptr ds:[eax+ecx*2-0xB5078F0]
;取出偏移为0x10虚拟寄存器的值 12345678
004DBC8B    81DF 04000000   sbb edi,0x4
004DBC91    0FB6C2          movzx eax,dl
004DBC94    c1c0 d1         rol eax,0xd1
004DBC97    898CC7 000090F2 mov dword ptr ds:[edi+eax*8-0xD700000],ecx
;将12345678存放在 0019FF6C
004DBC9E    58              pop eax            
004DBC9F    15 DDB40C00     adc eax,0xCB4DD
004DBCA4  - FFE0            jmp eax ;下一个代码块地址为 004FDC4E

```

```
004FDC4E    8D0CD5 BC1A2C5D lea ecx,dword ptr ds:[edx*8+0x5D2C1ABC]
004FDC55    8D8489 BC1BB0C0 lea eax,dword ptr ds:[ecx+ecx*4-0x3F4FE444]
004FDC5C    58              pop eax  
004FDC5D    59              pop ecx
004FDC5E  ^ E9 E0FEFFFF     jmp 123_vmp.004FDB43

```

```
004FDB43    B8 05E6A7EC     mov eax,0xECA7E605
004FDB48    8D8424 84000000 lea eax,dword ptr ss:[esp+0x84]
004FDB4F    E8 BFD4FEFF     call 123_vmp.004EB013

```

```
004EB013  ^\E9 8525F7FF     jmp 123_vmp.0045D59D

```

```
0045D59D    3BF8            cmp edi,eax
0045D59F    E8 0F12FFFF     call 123_vmp.0044E7B3

```

```
0044E7B3  ^\E9 4963FEFF     jmp 123_vmp.00434B01

```

```
00434B01    E8 7B510D00     call 123_vmp.00509C81

```

```
00509C81    8D6424 0C       lea esp,dword ptr ss:[esp+0xC]
00509C85    0F87 68000000   ja 123_vmp.00509CF3    ;永远跳转

```

```
00509CF3   /E9 A9290000     jmp 123_vmp.0050C6A1

```

```
0050C6A1  ^\FFE5            jmp ebp      ;执行下一条handler 004D55D9

```

上面这条 handler 就做了一件事，将 12345678 存放在 0019FF6C。我们再看第七条 handler

```
004D55D9    B9 260CB0F8     mov ecx,0xF8B00C26
004D55DE    8BC1            mov eax,ecx
004D55E0    0FB65426 FF     movzx edx,byte ptr ds:[esi-0x1]
;从伪代码读取1字节到edx
004D55E5    0FA3C8          bt eax,ecx
004D55E8    C0F0 03         sal al,0x3
004D55EB    32D3            xor dl,bl ;开始解密虚拟寄存器偏移
004D55ED    66:2BC1         sub ax,cx
004D55F0    2AC4            sub al,ah
004D55F2    0f98c1          sets cl
004D55F5    D0C2            rol dl,1
004D55F7    50              push eax
004D55F8    E9 FD460100     jmp 123_vmp.004E9CFA

```

```
004E9CFA    874C24 00       xchg dword ptr ss:[esp],ecx
004E9CFE    FECA            dec dl
004E9D00    98              cwde
004E9D01    58              pop eax
004E9D02    F6DA            neg dl
004E9D04    D0CA            ror dl,1   ;解密虚拟寄存器偏移完成
004E9D06    8D8400 3FD0A31E lea eax,dword ptr ds:[eax+eax+0x1EA3D03F]
004E9D0D    32DA            xor bl,dl  ;生成新的密钥
004E9D0F    0FABC1          bts ecx,eax
004E9D12    03D4            add edx,esp
;算出偏移为0x20虚拟寄存器地址 vm_ecx
004E9D14    8B8C42 822FF8DF mov ecx,dword ptr ds:[edx+eax*2-0x2007D07E]
;取出vm_ecx的值
004E9D1B    8D90 3CF42154   lea edx,dword ptr ds:[eax+0x5421F43C]
004E9D21    898C38 BD17FCEF mov dword ptr ds:[eax+edi-0x1003E843],ecx
;将vm_ecx的值存放在19ff68
004E9D28    8D88 ABE89DB2   lea ecx,dword ptr ds:[eax-0x4D621755]
004E9D2E    8B8C06 BC17FCEF mov ecx,dword ptr ds:[esi+eax-0x1003E844]
;从伪代码读取4字节到ecx
004E9D35    33CB            xor ecx,ebx;开始解密handler地址相加数
004E9D37    c1c2 e2         rol edx,0xe2
004E9D3A    0FA3D0          bt eax,edx
004E9D3D    D1C1            rol ecx,1
004E9D3F    c0ea e6         shr dl,0xe6
004E9D42    F7D1            not ecx
004E9D44    0FB3C0          btr eax,eax
004E9D47    66:0FBBD0       btc ax,dx
004E9D4B    8D8C08 231162AC lea ecx,dword ptr ds:[eax+ecx-0x539DEEDD]
004E9D52    50              push eax
004E9D53    0FC9            bswap ecx ;解密handler地址相加数完成
004E9D55    52              push edx
004E9D56    81F2 2E859AE2   xor edx,0xE29A852E
004E9D5C    33D9            xor ebx,ecx  ;生成新的密钥
004E9D5E    66:99           cwd
004E9D60    038404 CD17FCEF add eax,dword ptr ss:[esp+eax-0x1003E833]
004E9D67    2AC2            sub al,dl
004E9D69    03E9            add ebp,ecx ;算出下一条handler地址 004D55D9
004E9D6B    03C2            add eax,edx
004E9D6D    0FB68C16 FBFFF1>movzx ecx,byte ptr ds:[esi+edx-0x720E0005]
;从伪代码读取1字节到ecx
004E9D75    50              push eax
004E9D76    99              cdq
004E9D77    FF4C24 08       dec dword ptr ss:[esp+0x8]
004E9D7B    83D6 FA         adc esi,-0x6
004E9D7E    FF4424 04       inc dword ptr ss:[esp+0x4]
004E9D82    50              push eax
004E9D83    32CB            xor cl,bl ;开始解密虚拟寄存器偏移
004E9D85    D0C9            ror cl,1
004E9D87    0FC14424 0C     xadd dword ptr ss:[esp+0xC],eax
004E9D8C    32CA            xor cl,dl
004E9D8E    c1f2 2b         sal edx,0x2b
004E9D91    c1c0 f9         rol eax,0xf9
004E9D94  ^ 0F85 F2F7F7FF   jnz 123_vmp.0046958C   ;永远跳转

```

```
0046958C    80F1 2F         xor cl,0x2F
0046958F    c0e2 e2         shl dl,0xe2
00469592    0f98c0          sets al
00469595    8D9490 2F0C1583 lea edx,dword ptr ds:[eax+edx*4-0x7CEAF3D1]
0046959C    FEC1            inc cl   ;解密虚拟寄存器偏移完成
0046959E    c1f2 58         sal edx,0x58
004695A1    10B414 040000D1 adc byte ptr ss:[esp+edx-0x2EFFFFFC],dh
004695A8    66:81D2 31A7    adc dx,0xA731
004695AD    32D9            xor bl,cl  ;生成新的密钥
004695AF    FEC8            dec al
004695B1    8D4C0C 10       lea ecx,dword ptr ss:[esp+ecx+0x10]
;算出偏移为0x3c虚拟寄存器地址 vm_edx
004695B5    F7D2            not edx
004695B7    8B8401 01F8DF93 mov eax,dword ptr ds:[ecx+eax-0x6C2007FF]
;取出vm_edx的值
004695BE    8D1455 3B0328DE lea edx,dword ptr ds:[edx*2-0x21D7FCC5]
004695C5    0FB6CA          movzx ecx,dl
004695C8    898439 21FFFFFF mov dword ptr ds:[ecx+edi-0xDF],eax
;将vm_edx的值存放在19ff64
004695CF    B8 A463BFD5     mov eax,0xD5BF63A4
004695D4    0FBAF0 0E       btr eax,0xE
004695D8    09840C 2EFFFFFF or dword ptr ss:[esp+ecx-0xD2],eax
004695DF    8DBC4F 4AFEFFFF lea edi,dword ptr ds:[edi+ecx*2-0x1B6]
004695E6    66:0FA3C8       bt ax,cx
004695EA    0FC18C4C 5EFEFF>xadd dword ptr ss:[esp+ecx*2-0x1A2],ecx
004695F2    59              pop ecx
004695F3    58              pop eax
004695F4    58              pop eax
004695F5    58              pop eax
004695F6    0F8E 47450900   jle 123_vmp.004FDB43   ;永远跳转

```

```
004FDB43    B8 05E6A7EC     mov eax,0xECA7E605
004FDB48    8D8424 84000000 lea eax,dword ptr ss:[esp+0x84]
004FDB4F    E8 BFD4FEFF     call 123_vmp.004EB013

```

```
004EB013  ^\E9 8525F7FF     jmp 123_vmp.0045D59D

```

```
0045D59D    3BF8            cmp edi,eax
0045D59F    E8 0F12FFFF     call 123_vmp.0044E7B3

```

```
0044E7B3  ^\E9 4963FEFF     jmp 123_vmp.00434B01

```

```
00434B01    E8 7B510D00     call 123_vmp.00509C81

```

```
00509C81    8D6424 0C       lea esp,dword ptr ss:[esp+0xC]
00509C85    0F87 68000000   ja 123_vmp.00509CF3   ;永远跳转

```

```
00509CF3   /E9 A9290000     jmp 123_vmp.0050C6A1

```

```
0050C6A1  ^\FFE5            jmp ebp     ;执行下一条handler 004D55D9

```

上面这条 handler 就做了两件事，将 vm_ecx 的值存放在 19ff68 和将 vm_edx 的值存放在 19ff64。我们再看第八条 handler ，第八条 handler 地址还是 004D55D9，这边我就不再分析了，记录一下操作， 将 vm_ebp 的值存放在 19ff60 和将 vm_esi 的值存放在 19ff5c。我们再看第九条 handler ，第九条 handler 地址还是 004D55D9，记录一下操作， 将 vm_eflags 的值存放在 19ff58 和将 vm_ebx 的值存放在 19ff54。我们再看第十条 handler ，第十条 handler 地址是 00494549

```
00494549    B9 B21F9650     mov ecx,0x50961FB2
0049454E    0FBFC1          movsx eax,cx
00494551    8DB431 4AE069AF lea esi,dword ptr ds:[ecx+esi-0x50961FB6];更新esi的值
00494558    8D9441 A6FD8913 lea edx,dword ptr ds:[ecx+eax*2+0x1389FDA6]
0049455F    66:0FABC8       bts ax,cx
00494563    8B9431 4EE069AF mov edx,dword ptr ds:[ecx+esi-0x50961FB2]
;从伪代码读取4字节到edx
0049456A    0F83 FE280700   jnb 123_vmp.00506E6E   ;永远跳转

```

```
00506E6E    33D3            xor edx,ebx ;开始解密handler地址相加数
00506E70    0FBAF9 3E       btc ecx,0x3E
00506E74    D1C2            rol edx,1
00506E76    c1c0 8e         rol eax,0x8e
00506E79    F7D2            not edx
00506E7B    66:33C1         xor ax,cx
00506E7E    E8 F220FEFF     call 123_vmp.004E8F75

```

```
004E8F75    0FBAE8 2B       bts eax,0x2B
004E8F79    8D944A F6B9399B lea edx,dword ptr ds:[edx+ecx*2-0x64C6460A]
004E8F80    898C0C 4EE069EF mov dword ptr ss:[esp+ecx-0x10961FB2],ecx
004E8F87    0FCA            bswap edx
004E8F89    86C1            xchg cl,al
004E8F8B    66:c18c0c 4ee06>ror word ptr ss:[ecx+esp+0xef69e04e],0x8b
004E8F94    33DA            xor ebx,edx  ;生成新的密钥
004E8F96    59              pop ecx   
004E8F97    03EA            add ebp,edx ;算出下一条handler地址 0049AC92
004E8F99    FEC1            inc cl
004E8F9B    8DB44E 7713D2DE lea esi,dword ptr ds:[esi+ecx*2-0x212DEC89]  ;更新esi的值
004E8FA2    0FB6D0          movzx edx,al
004E8FA5    99              cdq
004E8FA6    0FB68C0E BC0969>movzx ecx,byte ptr ds:[esi+ecx-0x1096F644] ;从伪代码读取4字节到ecx
004E8FAE    32CB            xor cl,bl ;开始解密虚拟寄存器偏移
004E8FB0  ^ E9 D644F6FF     jmp 123_vmp.0044D48B

```

```
0044D48B    0FCA            bswap edx
0044D48D    80F1 2B         xor cl,0x2B
0044D490    0BC2            or eax,edx
0044D492    0FBAF2 1A       btr edx,0x1A
0044D496    c0ca a4         ror dl,0xa4
0044D499    F6D1            not cl
0044D49B    80D9 3E         sbb cl,0x3E
0044D49E    48              dec eax
0044D49F    F6D1            not cl
0044D4A1    0F8F 55140700   jg 123_vmp.004BE8FC   ;永远跳转

```

```
004BE8FC    c1c8 e6         ror eax,0xe6
004BE8FF    12CA            adc cl,dl    ;解密虚拟寄存器偏移完成
004BE901    32D9            xor bl,cl  ;生成新的密钥
004BE903    03CC            add ecx,esp
;算出偏移为0x4虚拟寄存器地址 vm_edi
004BE905    66:F7D0         not ax
004BE908    c1c0 a4         rol eax,0xa4
004BE90B    4A              dec edx
004BE90C    8B8401 E4670BBE mov eax,dword ptr ds:[ecx+eax-0x41F4981C]
;取出vm_edi的值
004BE913    0FB6CA          movzx ecx,dl
004BE916    8DBC0F FDFEFFFF lea edi,dword ptr ds:[edi+ecx-0x103]
004BE91D    4A              dec edx
004BE91E    89844F 02FEFFFF mov dword ptr ds:[edi+ecx*2-0x1FE],eax
;将vm_edi的值存放在19ff50
004BE925    8D840A 099B12C6 lea eax,dword ptr ds:[edx+ecx-0x39ED64F7]
004BE92C    E8 B3D5FEFF     call 123_vmp.004ABEE4

```

```
004ABEE4    0f90c0          seto al
004ABEE7    5A              pop edx        
004ABEE8    0F8C 551C0500   jl 123_vmp.004FDB43   ;永远跳转

```

```
004FDB43    B8 05E6A7EC     mov eax,0xECA7E605
004FDB48    8D8424 84000000 lea eax,dword ptr ss:[esp+0x84]
004FDB4F    E8 BFD4FEFF     call 123_vmp.004EB013

```

```
004EB013  ^\E9 8525F7FF     jmp 123_vmp.0045D59D

```

```
0045D59D    3BF8            cmp edi,eax
0045D59F    E8 0F12FFFF     call 123_vmp.0044E7B3

```

```
0044E7B3  ^\E9 4963FEFF     jmp 123_vmp.00434B01

```

```
00434B01    E8 7B510D00     call 123_vmp.00509C81

```

```
00509C81    8D6424 0C       lea esp,dword ptr ss:[esp+0xC]
00509C85    0F87 68000000   ja 123_vmp.00509CF3   ;永远跳转

```

```
00509CF3   /E9 A9290000     jmp 123_vmp.0050C6A1

```

```
0050C6A1  ^\FFE5            jmp ebp     ;执行下一条handler 0049AC92

```

上面这条 handler 就干了一件事，将 vm_edi 的值存放在 19ff50 ，我们接着看第十一条 handler

```
0049AC92    8BE7            mov esp,edi ;修改esp为edi的值，为恢复寄存器做准备
0049AC94    5F              pop edi   ;恢复edi
0049AC95    5B              pop ebx   ;恢复ebx
0049AC96    B8 8ECD941D     mov eax,0x1D94CD8E
0049AC9B    9D              popfd     ;恢复eflags
0049AC9C    5E              pop esi   ;恢复esi 
0049AC9D    0FB6C8          movzx ecx,al
0049ACA0    98              cwde
0049ACA1    5D              pop ebp   ;恢复ebp
0049ACA2    8D14CD 3CE01F73 lea edx,dword ptr ds:[ecx*8+0x731FE03C]
0049ACA9    E9 C8340400     jmp 123_vmp.004DE176

```

```
004DE176    87CA            xchg edx,ecx
004DE178    5A              pop edx ;恢复edx
004DE179    51              push ecx
004DE17A    8B8C0C 581BE08C mov ecx,dword ptr ss:[esp+ecx-0x731FE4A8] ;恢复ecx
004DE181    864424 01       xchg byte ptr ss:[esp+0x1],al
004DE185    874424 00       xchg dword ptr ss:[esp],eax
004DE189  ^ E9 C37BF6FF     jmp 123_vmp.00445D51

```

```
00445D51    8B8404 5C71E08C mov eax,dword ptr ss:[esp+eax-0x731F8EA4]
;这条指令执行完，eax = 12345678。这就模拟了第一条vm的指令
00445D58    68 B5BFAB16     push 0x16ABBFB5
00445D5D    8D6424 10       lea esp,dword ptr ss:[esp+0x10]
00445D61    C3              retn
;离开虚拟机
;同时也是模拟 第二条我们vm的指令 jmp 00415035

```

到此为止全部代码分析完成了，同时也是 VmProtect 从低版本到高版本的代码虚拟化分支分析系列全部结束了。

[[培训] 传播安全知识、拓宽行业人脉——看雪讲师团队等你加入！](https://bbs.kanxue.com/thread-275828.htm)

[#调试逆向](forum-4-1-1.htm) [#VM 保护](forum-4-1-4.htm)

上传的附件：

*   [vmp.rar](javascript:void(0);) （890.46kb，14 次下载）