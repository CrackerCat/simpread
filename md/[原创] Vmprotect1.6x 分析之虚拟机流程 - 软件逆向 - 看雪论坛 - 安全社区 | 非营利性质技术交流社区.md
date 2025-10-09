> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-288594.htm)

> [原创] Vmprotect1.6x 分析之虚拟机流程

相信大家对于 Vmprotect 虚拟机的逆向一定非常头痛吧，尤其是最新版本，我去官网看了一下已经到了 3.96 了，如果我们直接去分析他，那无疑是螳臂当车，杯水车薪，根本是不可能的事情了，经过多年的发展，已经发生了翻天覆地的变化了，在准备材料的过程中，我也浏览了早期大神们发的很多文章，而我介绍的这个版本大概是在 2010 年前的，具体时间没有去查，属于是初代版本的，而我之所以发这篇文章，其实我也是菜鸟，发出来和大家一起交流共同成长，可能会出现错误，老鸟勿喷。Ok，废话不多说。步入正题。

我要讲的内容大概有是 Vmp1.2 从到 1.6 的改动，为什么选择 1.6 呢，那是因为 1.64 版本开始加入了反调试，所以选择这个版本比较具有代表性，还有一个原因就是，他们中间的版本网上不好找，而且相邻的版本之间改动的并不多，没必要每个版本都去分析。相比于 1.2 版本 1.6 版本，虚拟寄存器已经放在了堆栈里面了，1.2 版本是放在 vmp 区段，这是一个比较有意思的变动。有意思的是我找的 1.64 版本是个 demo 版本，很多功能被阉割了，连代码混淆都没有，这恰巧给我分析带来了参照，所以，我将结合 1.63 和 1.64 版本进行分析讲解，我第一次拿到 1.63 版本给程序加密后，用 OD 调试的时候发现，看起来感觉这还是 vmp 吗？怎么跟我分析的 1.2 版本不大一样，一眼看上去像天书。没关系，接下来我会慢慢的给大家分析出来。这篇帖子如果对于初学者来说还是有一定难度的，不过没关系我出过一部关于 vmprotect 逆向入门的课程，讲的非常详细主要就是围绕 vmp1.2 版本的深度讲解，https://www.kanxue.com/book-section_list-200.htm 大家可以去看看，如果手头不宽裕的也可以自己去研究，其实并不难，想开飞机的可以花点小钱。

我们找到 vmp1.64 目录下 HelloASM.vmp1.exe, 这个程序经过 vm 过的，设置选项选择最快速度，然后把检测调试器勾去掉，vm 的代码我们在程序入口点选择 5 个 nop，如下所示

```
nop
nop
nop
nop
nop

```

这 5 个 nop 会被忽略，根据我以前分析 1.2 的经验，他还是会生成把当前寄存器入虚拟堆栈和出虚拟堆栈的伪代码并执行，之所以采用最简单的加密也是为了排除不必要的额外分析，然后生成的这个程序，把他拖到 OD 我们跟踪看下，之前的 5 个 nop，被替换成了 jmp 如下代码

```
00401000 > $- E9 7D470000   jmp HelloASM.00405782

```

```
00405782   > \68 90574000   push HelloASM.00405790 ;伪代码地址
00405787   .  E8 5AF9FFFF   call HelloASM.004050E6

```

```
004050E6 56            push esi                               
004050E7 53            push ebx
004050E8 50            push eax
004050E9 53            push ebx
004050EA 51            push ecx                                
004050EB 9C            pushfd
004050EC 52            push edx                                
004050ED 57            push edi                                
004050EE 55            push ebp
004050EF 68 00000000   push 0x0
004050F4 8B7424 2C     mov esi,dword ptr ss:[esp+0x2C];取出伪代码地址
004050F8 89E5          mov ebp,esp;存放栈顶指针，为下一条做准备
004050FA 81EC C0000000 sub esp,0xC0;开辟虚拟寄存器堆栈空间
00405100 89E7          mov edi,esp;edi;存放虚拟寄存器的基地址
00405102 0375 00       add esi,dword ptr ss:[ebp];伪代码重定位
00405105 8A06          mov al,byte ptr ds:[esi] ;取出伪代码一个字节
00405107 0FB6C0        movzx eax,al
0040510A 8D76 01       lea esi,dword ptr ds:[esi+0x1];伪代码指针加1
0040510D FF2485 6A5240 jmp dword ptr ds:[eax*4+0x40526A];执行对应handler

```

接着单步我们来看下，如何将当前寄存器值压入虚拟堆栈的

```
0040522F  80E0 3C        and al,0x3C ;计算偏移
00405232  8B55 00        mov edx,dword ptr ss:[ebp];取出前面入栈的寄存器
00405235  83C5 04        add ebp,0x4 ;每个寄存器占堆栈4字节大小
00405238  891407         mov dword ptr ds:[edi+eax],edx;存放到对应虚拟堆栈     
0040523B  E9 C5FEFFFF    jmp HelloASM.00405105;跳回循环压入虚拟堆栈

```

跟踪发现，上面这条指令一共执行了 12 次，对应的伪代码如下

```
00405791  09 01 11 25 15 39 35 3D 21 29 1D 0C             

```

因为我 vm 的代码是 nop，编译的时候其实是会被忽略的，如果是其他代码，执行到这边，就是真正开始执行被 vm 的代码了，既然这边没有东西需要执行，那就进入收尾阶段就是，取出返回地址完了之后把堆栈上的所有虚拟寄存器，全部赋值到堆栈，我们可以继续看下代码是怎么样的

```
004051D0  |> \80E0 3C       |and al,0x3C ;计算偏移
004051D3  |.  8B1407        |mov edx,dword ptr ds:[edi+eax];从虚拟寄存器取出值
004051D6  |.  83ED 04       |sub ebp,0x4;每个寄存器占堆栈4字节大小
004051D9  |.  8955 00       |mov dword ptr ss:[ebp],edx ; 存放到对应的堆栈     
004051DC  |.  E9 E7040000   |jmp HelloASM.004056C8;

```

接着我们看下 jmp HelloASM.004056C8 的代码

```
004056C8  |>  8D47 50       lea eax,dword ptr ds:[edi+0x50]
004056CB  |. |39C5          cmp ebp,eax
004056CD  |.^|0F87 32FAFFFF ja HelloASM.00405105
004056D3  |. |89E2          mov edx,esp
004056D5  |. |8D4F 40       lea ecx,dword ptr ds:[edi+0x40]
004056D8  |. |29D1          sub ecx,edx
004056DA  |. |8D45 80       lea eax,[local.32]
004056DD  |. |80E0 FC       and al,0xFC
004056E0  |. |29C8          sub eax,ecx                             
004056E2  |. |89C4          mov esp,eax
004056E4  |. |9C            pushfd
004056E5  |. |56            push esi                                
004056E6  |. |89D6          mov esi,edx
004056E8  |. |8D7C08 C0     lea edi,dword ptr ds:[eax+ecx-0x40]
004056EC  |. |57            push edi
004056ED  |. |89C7          mov edi,eax
004056EF  |. |FC            cld
004056F0  |. |F3:A4         rep movs byte ptr es:[edi],byte ptr ds:[>
004056F2  |. |5F            pop edi                                 
004056F3  |. |5E            pop esi                                 
004056F4  |. |9D            popfd
004056F5  |.^|E9 0BFAFFFF   jmp HelloASM.00405105

```

上面这段代码比较长，我使用的这个 demo，执行到 ja HelloASM.00405105，就跳转了，我猜测应该是执行某些特殊的 handler 才需要执行后面那些代码吧，他们最终都是会跳回 00405105 继续读取执行伪代码，在这里我稍微卡了一下，没必要把这里也弄的那么清楚，这里只是一个小小的细节而已，OK，我们继续，执行到这边我们发现 esp 的值一直都没变，从开始执行 handler 之后 esp 这个寄存器就没有用到了，倒是 epb 反复的被用到，想想还蛮复杂的，有很多细节我们不必太在意。我们刚刚执行到这边，结果其实就是这两句

```
004051D6  |.  83ED 04       |sub ebp,0x4
004051D9  |.  8955 00       |mov dword ptr ss:[ebp],edx

```

ebp 在这里可以比作 esp，这时候 edx 的值是 0，其实上面两句是不是相当于 push edx，接着我们再往下执行，又来到了 004056C8 这里，在这边这一部分代码可以忽略，我们  
单步到 0040510D ，去看看下一条 handler，

```
00405772  |> \8B06          mov eax,dword ptr ds:[esi];从伪代码读取4字节
00405774  |.  83ED 04       sub ebp,0x4;
00405777  |.  8945 00       mov dword ptr ss:[ebp],eax;存放读到的结果
0040577A  |.  83C6 04       add esi,0x4;伪代码指针加4
0040577D  \.^ E9 46FFFFFF   jmp HelloASM.004056C8;又是跳到004056C8忽略

```

由于是使用 demo 版本的 vmp 编译的所以，数据都没有加密啥的，所以我们可以在伪代码看到 401005 这个地址，如果是 push xxxxxxxx，我们甚至可以看到 xxxxxxx，当然正式版不可能看到的，不过这方便了我们分析，我们再看下一条 handler

```
004056FA  |> \8B45 00       mov eax,dword ptr ss:[ebp];取出地址4010005          
004056FD  |.  0145 04       add dword ptr ss:[ebp+0x4],eax;和前面的0相加
00405700  |.  9C            pushfd;执行算术指令都要入栈标志寄存器
00405701  |.  8F45 00       pop dword ptr ss:[ebp];存放到ebp
00405704  |.^ E9 FCF9FFFF   jmp HelloASM.00405105;

```

上面是一个加法的 handler，用于重定位，算出返回地址的时候同时已经把结果存放到堆栈里面了。接着我们看下一条 handler

```
0040522F  |> \80E0 3C       |and al,0x3C
00405232  |.  8B55 00       |mov edx,dword ptr ss:[ebp]
00405235  |.  83C5 04       |add ebp,0x4
00405238  |.  891407        |mov dword ptr ds:[edi+eax],edx
0040523B  |.^ E9 C5FEFFFF   |jmp HelloASM.00405105

```

上面这条 handler 是把标志寄存器，存放到对应的虚拟标志寄存器，我们接着看下一条 handler，

```
004051D0  |> \80E0 3C       |and al,0x3C
004051D3  |.  8B1407        |mov edx,dword ptr ds:[edi+eax]         
004051D6  |.  83ED 04       |sub ebp,0x4
004051D9  |.  8955 00       |mov dword ptr ss:[ebp],edx
004051DC  |.  E9 E7040000   |jmp HelloASM.004056C8

```

上面是把虚拟寄存器的值全部弹出到堆栈，总共执行 10 次，下面是对应的伪代码

```
004057A4  20 3C 34 38 14 24 10 00 08 0C                     <48$..

```

这就是前面把堆栈上的值入虚拟寄存器相反的操作了，最后执行一个返回 handler

```
0040521A  |> \89EC          |mov esp,ebp
0040521C  |.  59            |pop ecx                                
0040521D  |.  5D            |pop ebp                                
0040521E  |.  5F            |pop edi                                
0040521F  |.  5A            |pop edx                                
00405220  |.  9D            |popfd
00405221  |.  59            |pop ecx                                
00405222  |.  5B            |pop ebx                                
00405223  |.  58            |pop eax                                
00405224  |.  5E            |pop esi                                
00405225  |.  5E            |pop esi                                
00405226  |.  C3            |retn

```

执行完 retn 后就回到了 401005，以上就是 1.64 最简单的虚拟机的执行过程了。我们发现和 1.2 比把虚拟寄存器放在堆栈上，确实让 handler 变得更加复杂化，为什么刚开始把虚拟寄存器放在 vmp 区段，应该是刚开始开发没那么多精力吧，什么东西都是摸石头过河，没办一下子，做到完美的原因吧。要是这么简单，确实 vmp 也不难，于是作者肯定要在每个指令上下功夫，垃圾指令加膨胀。1.63 版本的我刚拿到分析的时候确实有点懵，感觉就是一堆看不懂的指令外加各种跳转，确实这么一搞很有效果，反正我第一眼看了，在想，完了这还怎么分析，都看不懂，不过猪鼻子插葱，再怎么也不可能变成大象，你们说对不对，比如说，伪代码地址，和计算 handler 有关的代码比如 eax*4 + 地址，等等都是能够找到的，在这个版本是这样的，后面版本作者想要做的更加隐蔽，也是有可能的

前面我们分析了 1.64demo 版的 vm 代码，接下来我们看看，使用同样的设置，在 1.63 编译后的在 OD 调试时候的样子，将 vmp1.63 目录下的 HelloASM.vmpa.exe 拖入 OD，

```
00401000 > $- E9 B86F0000   jmp HelloASM.00407FBD;一样的jmp

```

```
00407FBD    68 B83873D3     push 0xD37338B8;这边应该是加密后的伪代码地址
00407FC2    E8 F4EAFFFF     call HelloASM.00406ABB

```

```
00406ABB   \0F8B 98F2FFFF   jpo HelloASM.00405D59;干扰分析的垃圾代码
00406AC1    56              push esi         ;     不管他             
00406AC2    E8 2DEFFFFF     call HelloASM.004059F4;这个call起连接代码作用

```

```
004059F4    66:892C24       mov word ptr ss:[esp],bp;干扰分析的垃圾代码
004059F8    891C24          mov dword ptr ss:[esp],ebx;干扰分析的垃圾代码
004059FB    66:0FBEF0       movsx si,al;干扰分析的垃圾代码
004059FF    E8 9D0C0000     call HelloASM.004066A1;这个call起连接代码作用

```

```
004066A1    8DB2 AF475D2D   lea esi,dword ptr ds:[edx+0x2D5D47AF]
004066A7    890424          mov dword ptr ss:[esp],eax
004066AA    66:BE C8FB      mov si,0xFBC8
004066AE    9C              pushfd
004066AF    52              push edx                                
004066B0    893C24          mov dword ptr ss:[esp],edi              
004066B3    66:FFCE         dec si
004066B6    E8 91FFFFFF     call HelloASM.0040664C

```

```
0040664C    66:0FB6F0       movzx si,al
00406650    891C24          mov dword ptr ss:[esp],ebx
00406653    66:C1C6 03      rol si,0x3
00406657    52              push edx                                
00406658    66:C1DE 07      rcr si,0x7
0040665C    57              push edi                                
0040665D    66:0FBCF0       bsf si,ax
00406661    66:0FA5DE       shld si,bx,cl
00406665    890C24          mov dword ptr ss:[esp],ecx              
00406668    56              push esi
00406669    66:D1DE         rcr si,1
0040666C    66:19E6         sbb si,sp
0040666F    8D34F5 E3262F4A lea esi,dword ptr ds:[esi*8+0x4A2F26E3]
00406676    892C24          mov dword ptr ss:[esp],ebp
00406679    0FBCE9          bsf ebp,ecx                             
0040667C    66:89ED         mov bp,bp
0040667F    66:0FBDEF       bsr bp,di
00406683    21E6            and esi,esp
00406685    68 00000000     push 0x0
0040668A    E8 24EBFFFF     call HelloASM.004051B3

```

```
004051B3    66:0FBAE2 0F    bt dx,0xF
004051B8    8B7424 30       mov esi,dword ptr ss:[esp+0x30]
004051BC    66:D3DD         rcr bp,cl
004051BF    66:31E5         xor bp,sp
004051C2    66:FFCD         dec bp
004051C5    81C6 4BCA0CDA   add esi,0xDA0CCA4B
004051CB    66:C1E5 0A      shl bp,0xA
004051CF    66:C1E5 05      shl bp,0x5
004051D3    F7D6            not esi
004051D5    F8              clc
004051D6    85E5            test ebp,esp
004051D8    C1F5 0F         sal ebp,0xF
004051DB    81C6 3A833FAD   add esi,0xAD3F833A
004051E1    E8 1B0B0000     call HelloASM.00405D01

```

```
00405D01    0FCD            bswap ebp
00405D03    66:0FCD         bswap bp
00405D06    8DAD CCF46965   lea ebp,dword ptr ss:[ebp+0x6569F4CC]
00405D0C    F7D6            not esi;到这边伪代码地址解密出来了
00405D0E    5D              pop ebp                                 
00405D0F    66:0FBEE8       movsx bp,al
00405D13    66:890424       mov word ptr ss:[esp],ax
00405D17    8D6C24 04       lea ebp,dword ptr ss:[esp+0x4]
00405D1B    9C              pushfd
00405D1C    60              pushad
00405D1D    8D6424 28       lea esp,dword ptr ss:[esp+0x28]
00405D21    E9 F5050000     jmp HelloASM.0040631B

```

上面这几个代码使用一些垃圾无意义的指令让代码变复杂，干扰我们分析，可能还有一些别的操作，我们在看下一个代码块

```
0040631B    66:F7D3         not bx
0040631E    81EC C0000000   sub esp,0xC0;和1.64开头的一样
00406324    66:19C2         sbb dx,ax
00406327    66:29F7         sub di,si
0040632A    89E7            mov edi,esp;
0040632C    66:29EB         sub bx,bp
0040632F    89F3            mov ebx,esi                             
00406331    FEC8            dec al
00406333    F5              cmc
00406334    0375 00         add esi,dword ptr ss:[ebp];和1.64开头的一样
00406337    F6DA            neg dl
00406339    C0D8 06         rcr al,0x6
0040633C    66:0FBCD1       bsf dx,cx
00406340    0FC0C6          xadd dh,al
00406343    8A06            mov al,byte ptr ds:[esi];和1.64开头的一样
00406345    E8 20100000     call HelloASM.0040736A

```

```
0040736A    28D8            sub al,bl
0040736C    D0D6            rcl dh,1
0040736E    2C 54           sub al,0x54
00407370    F6D6            not dh
00407372    C0C8 07         ror al,0x7
00407375    66:0FB6D1       movzx dx,cl
00407379    66:0FBAE6 0E    bt si,0xE
0040737E    34 20           xor al,0x20
00407380    56              push esi                                
00407381    66:0FCA         bswap dx
00407384    0FADEA          shrd edx,ebp,cl
00407387    2C 3E           sub al,0x3E
00407389    FECA            dec dl
0040738B    28C6            sub dh,al
0040738D    0f93c2          setae dl
00407390    28C3            sub bl,al
00407392    F9              stc
00407393    66:0FCA         bswap dx
00407396    0FB6C0          movzx eax,al
00407399    FEC2            inc dl
0040739B    66:0FA4EA 02    shld dx,bp,0x2
004073A0    8B1485 BB664000 mov edx,dword ptr ds:[eax*4+0x4066BB];是不是很熟悉
004073A7    F9              stc
004073A8    9C              pushfd
004073A9    80FB 4E         cmp bl,0x4E
004073AC    F8              clc
004073AD    81F2 F2485483   xor edx,0x835448F2
004073B3    9C              pushfd
004073B4  ^ E9 F4E6FFFF     jmp HelloASM.00405AAD

```

```
00405AAD    C64424 04 AA    mov byte ptr ss:[esp+0x4],0xAA
00405AB2    E8 7E020000     call HelloASM.00405D35

```

```
00405D35    46              inc esi                                 
00405D36    F9              stc
00405D37    F7DA            neg edx
00405D39    38E0            cmp al,ah
00405D3B    66:0FA3C4       bt sp,ax
00405D3F    66:F7C3 C87D    test bx,0x7DC8
00405D44    81C2 C58E028B   add edx,0x8B028EC5
00405D4A    F5              cmc
00405D4B    C1C2 03         rol edx,0x3
00405D4E    F5              cmc
00405D4F    E9 D50F0000     jmp HelloASM.00406D29

```

```
00406D29    81F2 18A74290   xor edx,0x9042A718
00406D2F    52              push edx
00406D30    F9              stc
00406D31    F9              stc
00406D32    C1C2 1E         rol edx,0x1E
00406D35    60              pushad
00406D36    E9 4C080000     jmp HelloASM.00407587

```

```
00407587    4A              dec edx    ;handler的地址算出来了                             
00407588    F5              cmc
00407589    81C2 00000000   add edx,0x0
0040758F    9C              pushfd
00407590    9C              pushfd
00407591    9C              pushfd
00407592    895424 40       mov dword ptr ss:[esp+0x40],edx         
00407596    9C              pushfd
00407597    9C              pushfd
00407598    66:895424 08    mov word ptr ss:[esp+0x8],dx
0040759D    FF7424 48       push dword ptr ss:[esp+0x48]
004075A1    C2 4C00         retn 0x4C ;这时候堆栈就是handler的地址

```

以上就是未执行 handler 前的所有代码了，可以看到，通过解密各个数据然后加入了很多垃圾指令，让整个代码变得很没有可读性。接着往下执行到第一个 handler，

```
00405B43    F7C5 2D5361B1   test ebp,0xB161532D
00405B49    20F2            and dl,dh
00405B4B    8D14AD 732EEEED lea edx,dword ptr ds:[ebp*4-0x1211D18D]
00405B52    66:0FBBCA       btc dx,cx
00405B56    04 6F           add al,0x6F
00405B58    66:0FBDD5       bsr dx,bp
00405B5C    55              push ebp
00405B5D    34 CE           xor al,0xCE
00405B5F    66:F7D2         not dx
00405B62    66:0FBDD3       bsr dx,bx
00405B66    8D92 27ED469B   lea edx,dword ptr ds:[edx-0x64B912D9]
00405B6C    FEC0            inc al
00405B6E    66:0FBAEA 0C    bts dx,0xC
00405B73    F7D2            not edx                                 
00405B75    C0C0 06         rol al,0x6
00405B78    66:0FCA         bswap dx
00405B7B    F6DE            neg dh
00405B7D    66:11CA         adc dx,cx
00405B80    F5              cmc
00405B81    F6D8            neg al
00405B83    60              pushad
00405B84    80E0 3C         and al,0x3C;这个大家有印象吧，计算虚拟寄存器偏移
00405B87    8D96 13E4D2D0   lea edx,dword ptr ds:[esi-0x2F2D1BED]
00405B8D    66:0FB3C2       btr dx,ax
00405B91    8B55 00         mov edx,dword ptr ss:[ebp];取出要存放的虚拟寄存器的值
00405B94    84FD            test ch,bh
00405B96    E8 72020000     call HelloASM.00405E0D

```

```
00405E0D    83C5 04         add ebp,0x4
00405E10  ^ E9 2AF2FFFF     jmp HelloASM.0040503F

```

```
0040503F    E8 B8000000     call HelloASM.004050FC

```

```
004050FC    E8 E1240000     call HelloASM.004075E2

```

```
004075E2    891407          mov dword ptr ds:[edi+eax],edx;这边就存到虚拟寄存器了
004075E5    68 403EDBB9     push 0xB9DB3E40
004075EA    8D6424 34       lea esp,dword ptr ss:[esp+0x34]
004075EE  ^ E9 44EDFFFF     jmp HelloASM.00406337

```

```
00406337    F6DA            neg dl
00406339    C0D8 06         rcr al,0x6
0040633C    66:0FBCD1       bsf dx,cx
00406340    0FC0C6          xadd dh,al
00406343    8A06            mov al,byte ptr ds:[esi];又回到了读取伪代码的地方了
00406345    E8 20100000     call HelloASM.0040736A

```

上面这几个代码块就是执行第一个 handler 整个过程，这前面分析的 1.64 相比就是加了很多花指令，但是流程还不变的，所以我们可以推测出来，编译器在编译的时候先生成没有混淆的代码，也就是 1.64 分析的那样，然后进行变异膨胀加花指令等等。

接下来我们分析一下 1.64 的反调试，在帮助说明那边说了，v1.64 新增 调试器检测 和 VMWare 虚拟运行环境检测。果然这个版本强度直接上了一个档次，这边我就单独分析一下他的反调试，其他的分析思路都是一样的，我这边编译的时候设置跟前面一样，只不过检测调试器勾选择上，然后下拉框选择 user-mode，把编译好的 HelloASM.vmp2 拖到 OD 里面，

```
00408657 >  68 4D804000     push HelloASM.0040804D
0040865C    E8 AC020000     call HelloASM.0040890D
00408661    8A45 00         mov al,byte ptr ss:[ebp]
00408664    83ED 02         sub ebp,0x2
00408667    0045 04         add byte ptr ss:[ebp+0x4],al
0040866A    9C              pushfd
0040866B    8F45 00         pop dword ptr ss:[ebp]                  
0040866E    E9 73070000     jmp HelloASM.00408DE6
00408673    8A06            mov al,byte ptr ds:[esi]
00408675    8A0407          mov al,byte ptr ds:[edi+eax]
00408678    83ED 02         sub ebp,0x2
0040867B    66:8945 00      mov word ptr ss:[ebp],ax
0040867F    83EE FF         sub esi,-0x1
00408682    E9 5F070000     jmp HelloASM.00408DE6

```

上面的代码前两句没有用 jmp xxxxxxxx，而是直接 push 和 call，因为这边原来的代码已经被加密了，不需要考虑空间的问题了，当执行权交给宿主程序的时候，这边会被还原，然后这边可以看到已经被 handler 给替换了，这个和 1.2 版本的一模一样，我们继续跟进看看，

```
0040890D    53              push ebx
0040890E    55              push ebp
0040890F    56              push esi                                
00408910    52              push edx                                
00408911    50              push eax
00408912    52              push edx                                
00408913    9C              pushfd
00408914    51              push ecx                                
00408915    57              push edi                                
00408916    68 00000000     push 0x0
0040891B    8B7424 2C       mov esi,dword ptr ss:[esp+0x2C]
0040891F    89E5            mov ebp,esp
00408921    81EC C0000000   sub esp,0xC0
00408927    89E7            mov edi,esp
00408929    0375 00         add esi,dword ptr ss:[ebp]
0040892C    8A06            mov al,byte ptr ds:[esi]
0040892E    0FB6C0          movzx eax,al
00408931    8D76 01         lea esi,dword ptr ds:[esi+0x1]
00408934    FF2485 D5894000 jmp dword ptr ds:[eax*4+0x4089D5]

```

到这边就是和前面分析的一样进入 vm 了，我们可以在 00408934 下个断点，然后 F9 运行多次，他首先会先填充虚拟寄存器，也就是多次在 00408687 断下，然后就真正执行我们想要的代码了，接着解密返回地址，最后把虚拟寄存器的值给堆栈，再弹出到各自的寄存器中返回。按照这个思路我们其实只要分析，中间的那部分我们想要的代码就可以了，  
我们按 F9 运行多次在 00408687 断下后，接着在 00408899 断下，这个是从虚拟寄存器取出值放到 ebp 指向的堆栈里面，这个值显然是标志寄存器的值，应该没什么用，我们接着看下一条 handler，很显然是取值 handler，从伪代码取出 2 个字节的数据也是存放到 ebp 指向的堆栈，他的值是 0，再看下一条，和前面一样也是 0，占用空间却是 4 个字节，再下一条是，从虚拟寄存器中取出值放到 bp 指向的堆栈里面，和前面一样也是 0，占用空间 4 个字节，我们再看下一条，是个加法运算的 handler，是 0 和 0 相加，显然这个是耍流氓，干扰我们分析的，类似花指令，我们在看下一条，因为在虚拟机模拟算术运行过程中，第一条 handler 是计算，第二条 handler 就是把标志寄存器更新到对应的虚拟寄存器中，所以这一条忽略，分析到这边的时候大家是不是觉得很迷茫了，一个一个伪代码去分析已经不切实际，既然是分析检测调试器那部分伪代码，我们可以在 00408934 下断点，一直 F9 直到他弹出一个发现调试器的消息框，这时候再看下他的伪代码指针也就是 esi 的值，我是边按住 F9 边录屏，期间可以看到很多东西比如检测调试器的函数和提示内容等等，这些内容的解密都是在虚拟机里面进行的，我按住 F9 的时间也挺长的，编译生成的伪代码的量也挺多的，虽然加密只是 5 个 nop 指令，不像前面分析的伪代码量不多，但是反调试也在里面，这就相当于编译器私自加入一段反调试指令然后又 vm 了， 经过我多次的调试观察发现，他总共对三个地方进行了处理第一个就是 int3 异常改变 EIP，我们调试的时候就会被调试器捕捉，导致无法继续，

```
00407155    CC              int3;通过设置异常走异常处理函数00408050   
00407156    9D              popfd
00407157    68 4D754000     push HelloASM.0040754D
0040715C    E8 761C0000     call HelloASM.00408DD7

```

```
00408050    68 90844000     push HelloASM.00408490
00408055    E8 7D0D0000     call HelloASM.00408DD7

```

上面的处理方法，我们可以使用 StrongOD，把 Skip Some Exceptions 勾上，就可以过了，然而另外两个其实就是检测调试器的函数了，一个是 CheckRemoteDebuggerPresent，另外一个是 IsDebuggerPresent。这边我只拿一个出来分析，由于 StrongOD 第一个就是针对 IsDebuggerPresent 反调试的，为了方便分析我们把第一个勾勾上，这样我们就分析 CheckRemoteDebuggerPresent，这个反调试流程。我们在 CheckRemoteDebuggerPresent 下个断点，运行之后断下，在 esp+8 位置就是存放结果的变量，如果检测到调试器，那就为真，这是个布尔类型的变量，我们要观察他这个变量是如果影响程序的流程的，我们记住这个变量的地址是 0019FF50，ctl+f9 执行到返回，再单步

```
00407440    68 4D794000     push HelloASM.0040794D
00407445    E8 C3140000     call HelloASM.0040890D

```

来到了上面的代码，我们可以看到来到了下一个流程块，每个流程块的虚拟寄存器的位置是不一样的，我们接着一直单步走，接下来又是和刚开始一样，填充虚拟寄存器，执行关键代码，计算返回地址，虚拟寄存器全部恢复到堆栈，再填充真实寄存器，返回执行下一个代码块，这里我们直接在 004086F3 下个断点，这个 handler 是取出 esi 一个字节，这边是 0，然后存放到 ebp 指向的堆栈，然后下一条 handler 就是把存放反调试结果的地址 0019FF50 取出来，存放到 ebp 指向的堆栈，我们再看下一条 handler，这边是取出 0019FF50 的值再放到 ebp 指向的堆栈也就是 0019F74C，我们再看下一条 handler，这里是把 0019F74C 这个地址存放到 0019F748，再下一条是，把反调试结果的值存放到 0019F74A，注意这边存放的大小是 2 个字节，我们再看下一条，

```
0040878B    66:8B45 00      mov ax,word ptr ss:[ebp];ax = 反调试结果
0040878F    66:8B55 02      mov dx,word ptr ss:[ebp+0x2];dx = 反调试结果
00408793    F6D0            not al
00408795    F6D2            not dl
00408797    83ED 02         sub ebp,0x2
0040879A    20D0            and al,dl
0040879C    66:8945 04      mov word ptr ss:[ebp+0x4],ax
004087A0    9C              pushfd
004087A1    8F45 00         pop dword ptr ss:[ebp]                  
004087A4    E9 3D060000     jmp HelloASM.00408DE6

```

上面的代码执行之后，后面还有很多类似的运算，就一句话，简单的问题复杂化，我先是按住 F9 录屏分两次一次是修改 CheckRemoteDebuggerPresent 函数执行完后，他的第二个参数，也就存放检测结果的那个参数，一次不修改，一次修改为 0，0 就表示没有检测到调试器，录完屏观察 esi 的变化，找到两次运行比较 esi 出现变化的地方下内存访问断点，这样一直往前分析，这边有几个关键点说下，上面那个代码是第一次变形，这个第一次指的是，输入的是原始数据也就是反调试结果这个布尔类型的值，变形就是通过某种运算比如说上面的运算，得到两个数据一个是 ax 的值，还有一个是运算后标志寄存器的状态，这两个数据接着作为下一次运算的输入值，一环扣一环，全部分析整个过程没有意义，当然我就这样描述感觉还不过，细节好像少了，这边我就在说下几个关键的地方，当我们在 00407980 下个内存访问断点，我们看这条 handler

```
00408E47    8B06            mov eax,dword ptr ds:[esi]              
00408E49    83ED 04         sub ebp,0x4
00408E4C    8945 00         mov dword ptr ss:[ebp],eax
00408E4F    8D76 04         lea esi,dword ptr ds:[esi+0x4]
00408E52  ^ E9 8FFFFFFF     jmp HelloASM.00408DE6

```

上面这条 handler 就是从伪代码中取出 esi 后面被替换的值，执行后 ebp 指向的堆栈值 0040813B，然后执行下一条 handler，其实下一条 handller，和刚刚执行的是同一个 handler，这次是把 00407664 存放在 ebp 指向的堆栈值，这两个相邻的堆栈地址，就是反调试结果的两个 esi 被替换的值，既然两个值都给他取出来了，那么下面就要通过判断决定使用哪个值了，我们在 00407996 下个内存访问断点，这时候的 handler 我们看下，

```
00408820    8B45 00         mov eax,dword ptr ss:[ebp]
00408823    8A4D 04         mov cl,byte ptr ss:[ebp+0x4]
00408826    83ED 02         sub ebp,0x2
00408829    D3E8            shr eax,cl
0040882B    8945 04         mov dword ptr ss:[ebp+0x4],eax
0040882E    9C              pushfd
0040882F    8F45 00         pop dword ptr ss:[ebp]                  
00408832    E9 AF050000     jmp HelloASM.00408DE6

```

上面的第一条指令执行完，当检测结果布尔值为 0 时，eax = 40 ， 为 1 时，eax = 0 ，这个 eax 的值就是前面经过很多次变形后的结果，这条 handler 执行完，得到的 eax 就是个偏移，我们接着去看下一条指令，这条 handler 前面说了，更新状态寄存器的，很多时候可以忽略，我们再看下下条 handler，

```
00408E38    8B45 00         mov eax,dword ptr ss:[ebp]
00408E3B    0145 04         add dword ptr ss:[ebp+0x4],eax
00408E3E    9C              pushfd
00408E3F    8F45 00         pop dword ptr ss:[ebp]                  
00408E42  ^ E9 E5FAFFFF     jmp HelloASM.0040892C

```

这条 handler 就是把存放第一个 esi 的值与 eax 相加，eax 为 0，那就结果指向 0040813B，为 1，就指向 00407664，到现在就已经存放好将要拿来替换 esi 的值了，下面再看看是哪里替换了 esi 的值，我们再在 4079ad 下个内存访问断点，我们看下 handler

```
004087A9    8B75 00         mov esi,dword ptr ss:[ebp]             
004087AC    83C5 04         add ebp,0x4
004087AF    E9 75010000     jmp HelloASM.00408929

```

这是一条修改 esi 值的 handler，在这边两个结果就此分道扬镳，这其实就是在模拟条件跳转，在真实指令上只需要几行，然而在虚拟机上却要这么大费周章，以上就是 1.64 反调试分析的整个过程，多亏了有 demo 版本的帮助，分析起来还是相对比较轻松的，不能说轻松吧，都花了一个上下午，本来想睡个午觉的结果，没睡成，这个也终于弄完了，可以去干别的事情了，前面那些分析方法完全就是拿着一把钝了的刀去砍柴，真正去分析还得拿着好用的工具去。

最后我打算建一个 vmp 学习群，，一个人的能力确实有限，欢迎志同道合的人，互帮互助，一起交流共同成长。  
![](https://bbs.kanxue.com/upload/attach/202509/1004848_7DDPWMZHNF29UMT.webp)

[[培训] 传播安全知识、拓宽行业人脉——看雪讲师团队等你加入！](https://bbs.kanxue.com/thread-275828.htm)

最后于 5 天前 被阿强编辑 ，原因：

[#VM 保护](forum-4-1-4.htm)

上传的附件：

*   [vmp.rar](javascript:void(0);) （2.72MB，15 次下载）