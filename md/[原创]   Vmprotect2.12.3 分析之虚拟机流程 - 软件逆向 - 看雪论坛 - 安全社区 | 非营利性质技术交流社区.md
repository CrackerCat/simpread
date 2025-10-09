> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-288627.htm)

> [原创] Vmprotect2.12.3 分析之虚拟机流程

终于到了分析用 delphi 语言编写的 vmp 保护接近最后一个版本了，delphi 时代最后的狂欢了，我说的不知道有没有问题，但是我想表达的就是这个意思，这个版本可能是 2.x 的网上可以找到的最后一个版本了，代码结构相较于之前版本改动不算很大，所以说是最后的狂欢了，而支持分析 vmp 的插件貌似只支持到 2.08，以后版本就不支持了，这也给后续版本的 vmp，增加更加神秘的面纱了，然后他的下个版本就是 3.0 了，听说代码重构了，改用 C++ 写了，我上一篇文章写的是 vmp1.6 的分析，他后面有 1.7，1.8，等等，变化不大，我就不浪费时间了，OK，2.12.3 这个版本正好取到承上启下的作用，等分析完这个版本就可以去分析 3.0 了，想想还是有点小兴奋，感觉像打怪升级，现在最新版本也 3.9 几，当然我这么说肯定要被大神们笑话了，有时候学习就是这样，给自己一点动力，方能事半功倍。

我们把 VmProtect.2.12.3 目录下的 HelloASM.vmp1 拖到 OD 里面，这个文件是用 vmp 编译过的，然后说下一些设置，首先选项这一栏我选择了最快速度，然后加密的字节，仍旧是程序入口的 5 个 nop 指令，生成的 exe 文件大小是 12kb，就这大小而言还不算大，这边再补充一下上个文章中的一个知识点，因为后面需要用到，下面是 1.64 版本虚拟机入口执行时候的例子

```
004050E6  |$  56            push esi                           
004050E7  |.  53            push ebx
004050E8  |.  50            push eax
004050E9  |.  53            push ebx
004050EA  |.  51            push ecx                           
004050EB  |.  9C            pushfd
004050EC  |.  52            push edx                           
004050ED  |.  57            push edi                           
004050EE  |.  55            push ebp
004050EF  |.  68 00000000   push 0x0
004050F4  |.  8B7424 2C     mov esi,dword ptr ss:[esp+0x2C]    
004050F8  |.  89E5          mov ebp,esp
004050FA  |.  81EC C0000000 sub esp,0xC0

```

```
0019FF44   00000000  ;1   push 0x0
0019FF48   0019FF80  ;2   push ebp
0019FF4C   00401000  ;3   push edi 
0019FF50   00401000  ;4   push edx 
0019FF54   00000246  ;5   pusfd
0019FF58   00401000  ;6   push ecx  
0019FF5C   003C9000  ;7   push ebx
0019FF60   0019FFCC  ;8   push eax
0019FF64   003C9000  ;9    push ebx
0019FF68   00401000  ;10   push esi    
0019FF6C   0040578C  ;前面执行的CALL压入的返回地址  
0019FF70   00405790  ;伪代码地址  

```

我说一下上面的两个部分，第一部分是 1.64demo 版本虚拟机入口的部分代码，当 EIP 为 004050FA 时，这时候堆栈就是 1.64demo 版本第二部分的样子，这时 ESP = 0019FF44，等下我们用来做对比，接着回到我们的现在的 OD。

首先我们看下第一条指令，是一个 jmp 指令，和以前一样，

```
00401000 > $- E9 8E480000   jmp HelloASM.00405893

```

再往下看

```
00405893    9C              pushfd
00405894    E9 73010000     jmp HelloASM.00405A0C

```

上面这边已经不是 push A ， call B 了，在 1.6 版本这个 A 就是指加密了的伪代码地址，这个 pushfd 是占坑的作用，当执行到 00405894 的时候我们把当前 8 个寄存器和标志寄存器的值记录一下，如下所示

```
EAX 0019FFCC
ECX 00401000 
EDX 00401000
EBX 00228000
ESP 0019FF70
EBP 0019FF80
ESI 00401000
EDI 00401000
Eflag 00000246

```

我们再往下看

```
00405A0C    E8 00000000     call HelloASM.00405A11
00405A11    C74424 04 05594>mov dword ptr ss:[esp+0x4], 00405905
00405A19    882C24          mov byte ptr ss:[esp],ch;垃圾指令
00405A1C    66:C70424 EE70  mov word ptr ss:[esp],0x70EE;垃圾指令
00405A22    880424          mov byte ptr ss:[esp],al;垃圾指令
00405A25    C70424 2F1D0185 mov dword ptr ss:[esp],0x85011D2F;垃圾指令
00405A2C    9C              pushfd
00405A2D    8D6424 04       lea esp,dword ptr ss:[esp+0x4]
00405A31  ^ E9 C1E9FFFF     jmp HelloASM.004043F7

```

00405A0C 和 00405A11 这两处代码加上前面的代码，是不是和 push A ， call B 一样了呢。00405A2C 和 00405A2D 两条指令组合等于啥也没干，垃圾指令。根据 1.6 版本的经验，后面的代码就把一堆寄存器入栈了

```
004043F7    60              pushad
004043F8    9C              pushfd
004043F9    885C24 04       mov byte ptr ss:[esp+0x4],bl
004043FD    FF7424 14       push dword ptr ss:[esp+0x14];push ebx
00404401    f3:9c           rep pushfd;pushfd
00404403    8F4424 24       pop dword ptr ss:[esp+0x24]
00404407    9C              pushfd
00404408    C60424 2B       mov byte ptr ss:[esp],0x2B
0040440C    9C              pushfd
0040440D    9C              pushfd
0040440E    8D6424 30       lea esp,dword ptr ss:[esp+0x30]
00404412    E9 C1110000     jmp HelloASM.004055D8

```

```
004055D8    66:81FF 79E0    cmp di,0xE079
004055DD    51              push ecx                           
004055DE    60              pushad
004055DF    895424 1C       mov dword ptr ss:[esp+0x1C],edx    
004055E3    C0E1 05         shl cl,0x5
004055E6    8D8E 7B0A5051   lea ecx,dword ptr ds:[esi+0x51500A7B]
004055EC    894424 18       mov dword ptr ss:[esp+0x18],eax
004055F0    80C1 C2         add cl,0xC2
004055F3    E8 4EFAFFFF     call HelloASM.00405046

```

```
00405046    877C24 18       xchg dword ptr ss:[esp+0x18],edi   
0040504A    F6D0            not al
0040504C    0FB6C9          movzx ecx,cl
0040504F    895C24 14       mov dword ptr ss:[esp+0x14],ebx
00405053    66:09F7         or di,si
00405056    84F1            test cl,dh
00405058    897424 10       mov dword ptr ss:[esp+0x10],esi    
0040505C    66:F7D7         not di
0040505F    66:D3D7         rcl di,cl
00405062    897424 0C       mov dword ptr ss:[esp+0xC],esi     
00405066    66:C1C7 03      rol di,0x3
0040506A    66:C1FF 05      sar di,0x5
0040506E    66:0FBAF1 0F    btr cx,0xF
00405073    20F1            and cl,dh
00405075    876C24 08       xchg dword ptr ss:[esp+0x8],ebp
00405079    66:39F7         cmp di,si
0040507C    FF35 164C4000   push dword ptr ds:[0x404C16]
00405082    8F4424 04       pop dword ptr ss:[esp+0x4]         
00405086    66:D1C6         rol si,1
00405089    50              push eax
0040508A    66:0FACC7 02    shrd di,ax,0x2
0040508F    C74424 04 00000>mov dword ptr ss:[esp+0x4],0x0;重定位有关系的
00405097    66:0FC1FE       xadd si,di
0040509B    8B7424 34       mov esi,dword ptr ss:[esp+0x34];esi赋值 
0040509F    66:0FA5C5       shld bp,ax,cl
004050A3    66:0FADCF       shrd di,cx,cl
004050A7    66:29E5         sub bp,sp
004050AA    8D6C24 04       lea ebp,dword ptr ss:[esp+0x4]
004050AE    66:C1D7 03      rcl di,0x3
004050B2    81EC BC000000   sub esp,0xBC;开辟堆栈

```

上面这些代码是不是跟我们前面提到的 1.64 版本中，从 004050E6 到 004050FA 的代码干的事情一样，只是变得很不好分析了。其实我们可以不用去一个一个去跟踪上面的代码，我们只要在 004050B2 下个断点，运行到这边看下堆栈，和前面保存的寄存做个比较，就可以还原出他，原来的样子。以下就是执行到 004050B2 时，堆栈的样子，因为这边寄存器好几个值都是 401000，我就保证他们每个都出现一次这样写，

```
0019FF3C   0019FF33  ;
0019FF40   00000000  ;push 0
0019FF44   00000000  ;
0019FF48   0019FF80  ;push ebp
0019FF4C   00401000  ;push ecx    
0019FF50   00401000  ;push edx    
0019FF54   00228000  ;push ebx  
0019FF58   00401000  ;push esi  
0019FF5C   0019FFCC  ;push eax
0019FF60   00401000  ;push edi
0019FF64   00401000  ;push 重复
0019FF68   00000246  ;pushfd
0019FF6C   85011D2F  ;前面执行的CALL压入的返回地址的堆栈位置 
0019FF70   00405905  ;伪代码地址 

```

上面的和 1.64 的对比了一下，基本上相同，这时候 esp = 19ff3c，ebp = 0019FF40。为什么我在 0019FF40 后面注释 push 0 ，而不是在 0019FF44 注释呢，我是根据 004050C8 的指令推断的，执行到这条指令 ebp 等于 0019FF40，当然后面填充虚拟寄存器的时候也可以证明这一点。多出来没有注释的，现在还不知道干什么用，很有可能就是让伪代码执行的更复杂，我们再往下看

```
004050B2    81EC BC000000   sub esp,0xBC;开辟堆栈
004050B8    66:0FBAF9 07    btc cx,0x7
004050BD    D2D8            rcr al,cl
004050BF    89E7            mov edi,esp;edi存放虚拟寄存器基地址
004050C1    D2E0            shl al,cl
004050C3    66:0FBDC9       bsr cx,cx
004050C7    F5              cmc
004050C8    0375 00         add esi,dword ptr ss:[ebp];重定位有关系的

```

```
0040463E    8A06            mov al,byte ptr ds:[esi];取出伪代码
00404640    66:0FACC9 02    shrd cx,cx,0x2
00404645    F6D1            not cl
00404647    D2E9            shr cl,cl
00404649    0FB6C0          movzx eax,al
0040464C    80F1 55         xor cl,0x55
0040464F    8B0C85 1A4C4000 mov ecx,dword ptr ds:[eax*4+0x404C1A];计算handler地址,不过此时还是加密的
00404656    E9 81050000     jmp HelloASM.00404BDC

```

```
00404BDC    83EE FF         sub esi,-0x1;等效于inc esi

```

```
00404BF7    81F1 CE66B527   xor ecx,0x27B566CE;解密

```

```
00404C02    894C24 24       mov dword ptr ss:[esp+0x24],ecx;handler地址存放到堆栈   
00404C06    C60424 0E       mov byte ptr ss:[esp],0xE
00404C0A    FF7424 24       push dword ptr ss:[esp+0x24]; handler地址入栈      
00404C0E    C2 2800         retn 0x28 ;转到handler执行

```

上面这些分析下来，和 1.6，1.2 版本相比并没有本质的变化。以上就是还没执行 handler 前的分析过程，下面开始就是分析 handler 的执行

```
00404125    F5              cmc
00404126    0FC0C2          xadd dl,al
00404129    66:29CA         sub dx,cx
0040412C    66:0FBDD3       bsr dx,bx
00404130    8A06            mov al,byte ptr ds:[esi];取出虚拟寄存器偏移
00404132    66:81C2 F096    add dx,0x96F0
00404137    8B55 00         mov edx,dword ptr ss:[ebp];将先前入栈的寄存器给edx
0040413A    F5              cmc
0040413B    E8 13010000     call HelloASM.00404253

```

```
00404253    8D76 01         lea esi,dword ptr ds:[esi+0x1];等效于inc esi
00404256    84FF            test bh,bh
00404258    83C5 04         add ebp,0x4
0040425B    9C              pushfd
0040425C    60              pushad
0040425D    891438          mov dword ptr ds:[eax+edi],edx;填充虚拟寄存器的值
00404260    FF7424 08       push dword ptr ss:[esp+0x8]
00404264    68 58E5DA5D     push 0x5DDAE558
00404269    9C              pushfd
0040426A    8D6424 34       lea esp,dword ptr ss:[esp+0x34]
0040426E    E9 580E0000     jmp HelloASM.004050CB

```

```
004050CB    20E8            and al,ch
004050CD    E8 6BF5FFFF     call HelloASM.0040463D;继续循环读取伪代码执行

```

以上第一条 handler 的执行过程，依旧是填充虚拟寄存器的 handler，将 19ff40 上的值也就是 0 填充到了虚拟寄存，有变化的是，少了个 and al,0x3C 这个大家应该有印象吧，之前读取的伪代码有两个作用，一个是计算 handler 地址，另外一个作用就是解密出虚拟寄存器的偏移，这个版本因为 [eax*4 + handler 基址] 算出的是加密后的 handler，那么这个虚拟寄存器的偏移，就得另外存放在伪代码里面。使用了两个字节的伪代码，09 和 3C，虚拟寄存器的偏移是 0x3C，对应前面的 push 0。  
我们再看下一条 handler

```
0040439F    0f96c4          setbe ah
004043A2    66:0FBEC3       movsx ax,bl
004043A6    37              aaa
004043A7    8B06            mov eax,dword ptr ds:[esi];从伪代码读取4字节的数据
004043A9    E8 4C070000     call HelloASM.00404AFA

```

```
00404AFA    83ED 04         sub ebp,0x4
00404AFD    68 C5C5E646     push 0x46E6C5C5
00404B02  ^ E9 60FDFFFF     jmp HelloASM.00404867

```

```
00404867    8945 00         mov dword ptr ss:[ebp],eax
0040486A    9C              pushfd
0040486B    80FF A6         cmp bh,0xA6
0040486E    83C6 04         add esi,0x4;伪代码指针往后移动四个字节
00404871    9C              pushfd;垃圾指令
00404872    9C              pushfd;垃圾指令
00404873    60              pushad;垃圾指令
00404874    8D6424 34       lea esp,dword ptr ss:[esp+0x34];垃圾指令
00404878    E9 9A0C0000     jmp HelloASM.00405517

```

```
00405517    D2F8            sar al,cl
00405519    D2EC            shr ah,cl
0040551B    C0DC 03         rcr ah,0x3
0040551E    8D47 50         lea eax,dword ptr ds:[edi+0x50]
00405521    9C              pushfd
00405522    83EC FC         sub esp,-0x4
00405525  ^ 0F8F AEEDFFFF   jg HelloASM.004042D9

```

```
004042D9    E8 67FEFFFF     call HelloASM.00404145

```

```
00404145    9C              pushfd
00404146    A8 75           test al,0x75
00404148    39C5            cmp ebp,eax
0040414A    C64424 04 94    mov byte ptr ss:[esp+0x4],0x94
0040414F    9C              pushfd
00404150    882424          mov byte ptr ss:[esp],ah
00404153    60              pushad
00404154    8D6424 2C       lea esp,dword ptr ss:[esp+0x2C]
00404158    0F87 6D0F0000   ja HelloASM.004050CB

```

```
004050CB    20E8            and al,ch
004050CD    E8 6BF5FFFF     call HelloASM.0040463D;跳回继续循环

```

以上 handler 的一些注意点，我们分析一下，00404867 上的这条指令，执行后，堆栈上 19ff40 的值变成 AD7B4B1E，上一条 handler 就是将 19ff40 的值 0，填充到虚拟寄存器，00404874 上的这条指令相当于 add esp，0x34。作用就是平衡前面的三个 push 操作，所以可以将他们视作垃圾指令，同样 00405522 和 00405525 组合在一起也是没用的指令，接着我们在看下 00405522 和 00405525 组合，这两条指令有点迷惑性，分析之后这个跳转相当于 jmp，jg 指令要跳转需要满足 esp 比 - 0x4 大，很显然这个条件永远满足，我们继续看 00404158 这个 ja 指令，从他开始往前找影响标志位的指令就是 404148 的这个比较指令，因为 ebp 和 eax 的值都是固定的，所以这个跳转指令相当于 jmp，00404154 这条指令也是平衡前面的入栈，所以他和前面的几条都是垃圾指令，这条 handler 总体分析下来总结就是，好像啥也不做，用了很多垃圾指令，和条件跳转干扰我们分析。

我们再看下一条 handler

```
0040566A    0FB6C3          movzx eax,bl
0040566D    14 4C           adc al,0x4C
0040566F    D2C8            ror al,cl
00405671    8B45 00         mov eax,dword ptr ss:[ebp]
00405674  ^ E9 19FBFFFF     jmp HelloASM.00405192

```

```
00405192    F5              cmc
00405193    0145 04         add dword ptr ss:[ebp+0x4],eax
00405196  ^ E9 C8F2FFFF     jmp HelloASM.00404463

```

```
00404463    9C              pushfd
00404464    60              pushad
00404465  ^ E9 F2FEFFFF     jmp HelloASM.0040435C

```

```
0040435C    FF7424 20       push dword ptr ss:[esp+0x20]
00404360    8F45 00         pop dword ptr ss:[ebp]             
00404363    9C              pushfd
00404364    56              push esi                           
00404365    8D6424 2C       lea esp,dword ptr ss:[esp+0x2C]
00404369    E9 5D0D0000     jmp HelloASM.004050CB

```

```
004050CB    20E8            and al,ch
004050CD    E8 6BF5FFFF     call HelloASM.0040463D

```

上面这条 handler 和前面一条 handler 都是干扰我们分析的，00405193 这一条指令的 ebp+4 = 0019FF40，就是我们前面填充虚拟寄存器的那个堆栈地址，既然他们是没有用的 handler，那他们对应的伪代码是不是可以跳过，就是执行完第一条 handler 后回到执行 40463e 处的代码的时候我们把 esi 的值改为 0040590D，这样不就是跳过了那两条 handler 了呢, 我试过了不行，但是在 1.2 版本碰到干扰的伪代码可以直接修改跳过他，但是这边还是可以改的，还需要改 ebp，我们知道第一条 handler 执行完，回到 40463e 处代码的时候，ebp = 0019FF44，接着执行第二条，第三条 handler 完后，回到 40463e 处代码的时候，ebp = 0019FF40，所以说直接修改 esi 的值就会出错，这也是作者故意给我们挖的坑，经过分析第二条 handler 迷惑性很强，又是在伪代码处读取数据，其实他就一个作用将 ebp 的值减去 4，然后第三条 handler 可以直接跳过。

我们接着往下看第四条 handler，也就是第一条执行的那个 handler，使用了两个字节的伪代码，09 和 0C，虚拟寄存器的偏移是 0C，从他填充到虚拟寄存器值的存放的位置判断这条也是用作混淆的。

我们接着往下看第五条 handler，也就是第一条执行的那个 handler，使用了两个字节的伪代码，09 和 0C，虚拟寄存器的偏移是 0C，从他填充到虚拟寄存器值的存放的位置判断这条也是用作混淆的。

我们接着往下看第六条 handler，也就是第一条执行的那个 handler，使用了两个字节的伪代码，04 和 24，虚拟寄存器的偏移是 24，对应前面的 push ebp。

我们接着往下看第七条 handler，也就是第一条执行的那个 handler，使用了两个字节的伪代码，09 和 08，虚拟寄存器的偏移是 08，对应前面的 push ecx。

我们接着往下看第八条 handler，也就是第一条执行的那个 handler，使用了两个字节的伪代码，04 和 34，虚拟寄存器的偏移是 34，对应前面的 push edx。

我们接着往下看第九条 handler，也就是第一条执行的那个 handler，使用了两个字节的伪代码，04 和 28，虚拟寄存器的偏移是 28，对应前面的 push ebx。

我们接着往下看第十条 handler，也就是第一条执行的那个 handler，使用了两个字节的伪代码，72 和 00，虚拟寄存器的偏移是 0，对应前面的 push esi。

我们接着往下看第十一条 handler，也就是第一条执行的那个 handler，使用了两个字节的伪代码，09 和 10，虚拟寄存器的偏移是 10，对应前面的 push eax。

我们接着往下看第十二条 handler，也就是第一条执行的那个 handler，使用了两个字节的伪代码，04 和 04，虚拟寄存器的偏移是 04，对应前面的 push edi。

我们接着往下看第十三条 handler，也就是第一条执行的那个 handler，使用了两个字节的伪代码，09 和 14，虚拟寄存器的偏移是 14，对应前面的 push 重复。

我们接着往下看第十四条 handler，也就是第一条执行的那个 handler，使用了两个字节的伪代码，04 和 38，虚拟寄存器的偏移是 38，对应前面的 push eflag。

我们接着往下看第十五条 handler，也就是第一条执行的那个 handler，使用了两个字节的伪代码，09 和 2c，虚拟寄存器的偏移是 2c，对应前面的 85011D2F 。

我们接着往下看第十六条 handler，也就是第一条执行的那个 handler，使用了两个字节的伪代码，04 和 30，虚拟寄存器的偏移是 30，对应前面的伪代码地址 。

执行到这边虚拟寄存器已经填充完毕。我们看下分布

```
0019FE80  00 10 40 00 00 10 40 00 00 10 40 00 1E 4B 7B AD 
0019FE90  CC FF 19 00 00 10 40 00 00 00 00 00 00 00 00 00 
0019FEA0  00 00 00 00 80 FF 19 00 00 70 38 00 2F 1D 01 85 
0019FEB0  05 59 40 00 00 10 40 00 46 02 00 00 00 00 00 00 

```

这里我把数值替换成对应的寄存器

```
0019FE80     esi       edi      ecx      混淆
0019FE90     eax       重复    未填充    未填充
0019FEA0    未填充      ebp     ebx     85011D2F
0019FEB0   伪代码地址   edx     eflag    重定位0

```

上面的这个位置表中根据之前分析 1.2 版本经验推断，混淆，未填充 (可能不只这两种)，这些位置的虚拟寄存器，可以当做变量随便使用，其他虚拟寄存器, 只在模拟被 vm 的代码时候，用于保存计算结果。

我们接着看第十七条 handler，使用了两个字节的伪代码，C4 和 38

```
0040521D    80CE FE         or dh,0xFE
00405220    8A06            mov al,byte ptr ds:[esi]
00405222    D0CE            ror dh,1
00405224    21D2            and edx,edx                        
00405226    8B1438          mov edx,dword ptr ds:[eax+edi] ;读取eflag到edx
00405229    66:0FBAE0 09    bt ax,0x9
0040522E    83ED 04         sub ebp,0x4 ;执行完ebp = 19ff70
00405231    F5              cmc
00405232    60              pushad
00405233    E8 23000000     call HelloASM.0040525B

```

```
0040525B    8955 00         mov dword ptr ss:[ebp],edx  ; [19ff70] = eflag的值
0040525E    F8              clc
0040525F    60              pushad
00405260    886424 04       mov byte ptr ss:[esp+0x4],ah
00405264    83EE FF         sub esi,-0x1 ;等于 inc esi
00405267    66:C74424 08 0C>mov word ptr ss:[esp+0x8],0xAB0C ;垃圾指令
0040526E    885C24 08       mov byte ptr ss:[esp+0x8],bl ;垃圾指令
00405272    C64424 14 E6    mov byte ptr ss:[esp+0x14],0xE6 ;垃圾指令
00405277    8D6424 44       lea esp,dword ptr ss:[esp+0x44] ;垃圾指令
0040527B    E9 97020000     jmp HelloASM.00405517

```

```
00405517    D2F8            sar al,cl ;垃圾指令
00405519    D2EC            shr ah,cl ;垃圾指令
0040551B    C0DC 03         rcr ah,0x3;垃圾指令
0040551E    8D47 50         lea eax,dword ptr ds:[edi+0x50];给eax赋值，后面cmp指令
00405521    9C              pushfd
00405522    83EC FC         sub esp,-0x4
00405525  ^ 0F8F AEEDFFFF   jg HelloASM.004042D9;等于jmp

```

```
004042D9    E8 67FEFFFF     call HelloASM.00404145

```

```
00404145    9C              pushfd
00404146    A8 75           test al,0x75
00404148    39C5            cmp ebp,eax ;ebp永远比eax大
0040414A    C64424 04 94    mov byte ptr ss:[esp+0x4],0x94
0040414F    9C              pushfd
00404150    882424          mov byte ptr ss:[esp],ah
00404153    60              pushad
00404154    8D6424 2C       lea esp,dword ptr ss:[esp+0x2C]
00404158    0F87 6D0F0000   ja HelloASM.004050CB;等于jmp

```

```
004050CB    20E8            and al,ch
004050CD    E8 6BF5FFFF     call HelloASM.0040463D

```

所以上面这条 handler 也是用于混淆的 handler 了，没作用。

我们接着看第十八条 handler，也就是第二条执行的那个 handler，前面分析过了，执行之前 ebp = 19ff70，执行完之后，ebp=19ff6c，将读取到的 4 字节存放到 ebp 中。对应 5 个字节伪代码 0x1E,0x73,B6,0x1B,0x46。

我们接着看第十九条 handler，这边我就不把对应的代码全部贴出来了，前面分析过后，基本上哪些是垃圾指令，都能够一眼看出来了，这里就做一些分析记录就行了，这条 handler 执行之后，[0019FF68] = FFFFFEFF, ebp = 0019FF68 , 对应的 3 字节伪代码，0x23，0xff,0xfe。esi = 00405931。

我们接着看第二十条 handler，这条 handler 执行之后，[0019FF68] = FFFFFEFF, ebp = 0019FF68 , 对应的 3 字节伪代码，0x23，0xff,0xfe。esi = 00405931。

我们接着看第二十一条 handler，这条 handler 执行之后，[0019FF64] = 0x246, ebp = 0019FF64, 对应的 2 字节伪代码，0x10，0x38。esi = 00405933。这条是混淆用的

我们接着看第二十二条 handler，这条 handler 执行之后，[0019FF60] = 0x0019FF64, ebp = 0019FF60, 对应的 1 字节伪代码，0x4e。esi = 00405934。还看不出来干什么的

我们接着看第二十三条 handler，这条 handler 执行之后，[0019FF60] = 0x246, ebp = 0019FF60, 对应的 1 字节伪代码，0x42。esi = 00405935。这条也是混淆用的，分析到这边，发现前面全是搞混淆的，这边以后我加快分析，就不每一条执行的 handler 都记录了。

伪代码地址 405936 对应的 handler，将某个值填充到偏移为 18 的虚拟寄存器，我们前面做了个表格，知道这个虚拟寄存器是未填充的，所以我们可以大胆判断，这条也是混淆用的。

伪代码地址 405939 对应的 handler，这边很有意思，将虚拟标志寄存器的值 0x246，存放到偏移为 0x2c 的那个虚拟寄存器，这边应该是虚拟寄存器位置互换，位于偏移为 0x38 的标志虚拟寄存器就解放了，后面有多次对其进行垃圾数据的写入 。

伪代码地址 40593D 对应的 handler，开始计算 esi 的值，这里是读取伪代码的值， 期间还会多次读取伪代码的数据进行各种解密计算，其中穿插有很多干扰的 handler，和一些其他的环节，这边我就不一一分析，计算过程了。

伪代码地址 40597B 对应的 handler，这条将偏移为 0x3c 的那个虚拟寄存器值 0 ，存放到 0x19ff6c，  
我们再看下一条 handler，这条将偏移为 0x8 的那个虚拟寄存器值 0x0401000，存放到 0x19ff68，  
我们再看下一条 handler，这条将偏移为 0x4 的那个虚拟寄存器值 0x0401000，存放到 0x19ff64，  
我们再看下一条 handler，这条将偏移为 0x0 的那个虚拟寄存器值 0x0401000，存放到 0x19ff60，  
我们再看下一条 handler，这条将偏移为 0x2c 的那个虚拟寄存器值 0x00000246，存放到 0x0019FF5C，  
我们再看下一条 handler，这条将偏移为 0x10 的那个虚拟寄存器值 0x0019FFCC，存放到 0x0019FF58，  
我们再看下一条 handler，这条将偏移为 0x4 的那个虚拟寄存器值 0x0401000，存放到 0x0019FF54，  
我们再看下一条 handler，这条将偏移为 0x28 的那个虚拟寄存器值 0x0037C000 存放到 0x0019FF50  
(这边解释一下，这个为什么和刚开始执行时候的 ebx 不一样，因为每次运行程序断在入口点的时候 ebx 都不一样, 由于我调试的时候重新运行了程序)  
我们再看下一条 handler，这条将偏移为 0x24 的那个虚拟寄存器值 0x0019FF80，存放到 0x0019FF4C  
我们再看下一条 handler，这条将偏移为 0x8 的那个虚拟寄存器值 0x00401000，存放到 0x0019FF48  
我们再看下一条 handler，这条将偏移为 0x14 的那个虚拟寄存器值 0x00401000，存放到 0x0019FF44  
我们再看下一条 handler，这条将偏移为 0xC 的那个虚拟寄存器值 0xAD7B4B1E，存放到 0x0019FF40

我们总结一下上面的这几条虚拟寄存器值吐出到 ebp 指向的堆栈操作，前面因为我命名的时候是根据他们的值命名的，但是发现有好几个寄存器值都是一样的, 其实这边应该把他们的值先设置成不同的比较好一点, 都分析到这里了，只能硬着头皮继续分析, 一些虚拟寄存器名字可能会有张冠李戴的错误，遇到这个不用管它。

我们再看下一条 handler，这一条是条读取指令执行完，[19ff3c] = 5284B4E2，我们再看下一条，这一条加法指令执行完，[19ff40] = 0，我们再看下一条，这一条就是对偏移为 0x34 的那个虚拟寄存器写入 00000257，这是个标志寄存器的值，所以这里正好印证前面的分析。  
我们再看下一条，这条将偏移为 0x3c 的那个虚拟寄存器值 0x0，存放到 0x19ff3c,  
我们再看下一条，这条将偏移为 0x38 的那个虚拟寄存器值 0x00405899，这个 0x00405899 就是我们前面讲的经过多次解密计算后的将要改写 esi 值的值，这里偏移为 0x38 的那个虚拟寄存器保存了这个值，再存放到 0x19ff38, 等待使用。  
我们再看下一条 handler, 这条 handler 对应的伪代码地址 0x40599f，我们全面分析一下，

```
004047EE    66:0FCE         bswap si ;垃圾指令
004047F1    66:0FBCF7       bsf si,di ;垃圾指令
004047F5    9C              pushfd
004047F6    66:0FABD6       bts si,dx ;垃圾指令
004047FA    8B75 00         mov esi,dword ptr ss:[ebp] ;  这里给修改了esi,esi = 00405899
004047FD    66:0FBAE1 0D    bt cx,0xD
00404802    84DB            test bl,bl
00404804    F9              stc
00404805    66:0FBAE0 08    bt ax,0x8
0040480A    83C5 04         add ebp,0x4
0040480D    68 E70B3726     push 0x26370BE7
00404812    8D6424 08       lea esp,dword ptr ss:[esp+0x8]
00404816    E9 A6080000     jmp HelloASM.004050C1

```

```
004050C1    D2E0            shl al,cl
004050C3    66:0FBDC9       bsr cx,cx
004050C7    F5              cmc
004050C8    0375 00         add esi,dword ptr ss:[ebp] ;重定位，这里ebp存放了，偏移为0x3c的那个虚拟寄存器值0x0
004050CB    20E8            and al,ch
004050CD    E8 6BF5FFFF     call HelloASM.0040463D

```

上面这一条 handler 执行完，esi 变为 0x00405899。

我们接着看下一条 handler，这条是将偏移为 0x28 的那个虚拟寄存器 (对应 ebx) 值改为 0。

我们接着看下一条 handler，这条是从伪代码读取 4 字节的数据 AD7B4B1E。我们接着看伪代码地址为 4058A3 的 handler，这里是将 AD7B4B1E 存放在偏移为 0x1c 的那个虚拟寄存器中,

我们接着看下一条 handler，这条 handler 是将 0019FF44 的值 00401000，存放到偏移为 0xC 的那个虚拟寄存器, 我们前面推导出，0019FF44 保存的是偏移为 0x14 的那个虚拟寄存器 (对应 edx), 这边应该也是虚拟寄存器位置互换了，

我们接着看下一条 handler，这条 handler 是将 0019FF48 的值 00401000，存放到偏移为 0x10 的那个虚拟寄存器, 我们前面推导出，0019FF48 保存的是偏移为 0x8 的那个虚拟寄存器 (对应 ecx), 这边应该也是虚拟寄存器位置互换了

我们接着看下一条 handler，这条 handler 是将 0019FF4C 的值 0019FF80，存放到偏移为 0x18 的那个虚拟寄存器, 我们前面推导出，0019FF4C 保存的是偏移为 0x24 的那个虚拟寄存器 (对应 ebp), 这边应该也是虚拟寄存器位置互换了

我们接着看下一条 handler，这条 handler 是将 0019FF50 的值 003AE000，存放到偏移为 0x14 的那个虚拟寄存器, 我们前面推导出，0019FF50 保存的是偏移为 0x28 的那个虚拟寄存器 (对应 ebx), 这边应该也是虚拟寄存器位置互换了

我们接着看下一条 handler，这条 handler 是将 0019FF54 的值 0401000，存放到偏移为 0x0 的那个虚拟寄存器, 我们前面推导出，0019FF54 保存的是偏移为 0x4 的那个虚拟寄存器 (对应 edi), 这边应该也是虚拟寄存器位置互换了

我们接着看伪代码地址为 4058CD 对应的 handler，这条 handler 是将 0019FF58 的值 0019FFCC，存放到偏移为 0x24 的那个虚拟寄存器, 我们前面推导出，0019FF58 保存的是偏移为 0x10 的那个虚拟寄存器 (对应 eax), 这边应该也是虚拟寄存器位置互换了, 这边要说一下这条 handler 执行前，前面执行了好多干扰 handler，所做的事情完全一样但是就是没有意义。

我们接着看下一条 handler，这条 handler 是将 0019FF5C 的值 0x246，存放到偏移为 0x34 的那个虚拟寄存器, 我们前面推导出，0019FF5C 保存的是偏移为 0x2C 的那个虚拟寄存器 (对应新 eflag), 这边应该也是虚拟寄存器位置互换了，这里我解释一下这个 eflag，他本来的偏移是 0x38，后来有换到 0x2C，这次又换到了 0x34，挺有意思的。从执行伪代码地址 40597B 开始，虚拟寄存器的值相当于拷贝了一份到堆栈上，所以可以随便去互换各个虚拟寄存器的位置。

我们接着看下一条 handler，这条 handler 是将 0019FF60 的值 0401000，存放到偏移为 0x30 的那个虚拟寄存器, 我们前面推导出，0019FF60 保存的是偏移为 0x0 的那个虚拟寄存器 (对应 esi), 这边应该也是虚拟寄存器位置互换了

我们接着看下一条 handler，这条 handler 是将 0019FF64 的值 0401000，存放到偏移为 0x8 的那个虚拟寄存器, 我们前面推导出，0019FF64 保存的是偏移为 0x4 的那个虚拟寄存器 (对应 edi), 这边应该也是虚拟寄存器位置互换了

我们接着看下一条 handler，这条 handler 是将 0019FF68 的值 0401000，存放到偏移为 0x20 的那个虚拟寄存器, 我们前面推导出，0019FF68 保存的是偏移为 0x8 的那个虚拟寄存器 (对应 ecx), 这边应该也是虚拟寄存器位置互换了

我们接着看下一条 handler，这条 handler 是将 0019FF6C 的值 0，存放到偏移为 0x3C 的那个虚拟寄存器, 我们前面推导出，0019FF6C 保存的是偏移为 0x3C 的那个虚拟寄存器 (对应重定位 0)。

我们接着看下一条 handler，这条 handler 是将 0019FF70 的值 0x246，存放到偏移为 0x4 的那个虚拟寄存器, 我们前面知道，0x246 就是虚拟标志寄存器的值。

我们接着看伪代码地址为 4058e9 对应的 handler，这条 handler 是从伪代码中读取 4 个字节的值 401005，这个值使我们虚拟机返回的地址。

到这里我们可以更新一下虚拟寄存器的表格了，我是根据前面分析，和他的值去猜测的，可能会有矛盾的地方，直接忽略就行，如下所示

```
0019FE80     edi       eflag     空闲      edx
0019FE90     ecx       ebx       ebp     AD7B4B1E
0019FEA0     ecx       eax      重定位0    空闲
0019FEB0     esi       空闲      空闲     重定位0

```

我们再看下一条 handler，这条将偏移为 0x4 的那个虚拟寄存器 (对应 eflag) 值 0x0246，存放到 0x0019FF6C。

我们再看下一条 handler，这条将偏移为 0xC 的那个虚拟寄存器 (对应 edx) 值 0x00401000，存放到 0x0019FF68。

我们再看下一条 handler，这条将偏移为 0x8 的那个虚拟寄存器值 0x00401000，存放到 0x0019FF64。

我们再看下一条 handler，这条将偏移为 0x24 的那个虚拟寄存器 (对应 eax) 值 0x0019FFCC，存放到 0x0019FF60。

我们再看下一条 handler，这条将偏移为 0x30 的那个虚拟寄存器 (对应 esi) 值 0x00401000，存放到 0019FF5C。

我们再看下一条 handler，这条将偏移为 0x14 的那个虚拟寄存器 (对应 ebx) 值 0x00228000，存放到 0019FF58。

我们再看下一条 handler，这条将偏移为 0x38 的那个虚拟寄存器值 0x461BFFD3，存放到 0019FF54。

我们再看下一条 handler，这条将偏移为 0x10 的那个虚拟寄存器 (对应 ecx) 值 0x00401000，存放到 0019FF50。

我们再看下一条 handler，这条将偏移为 0x18 的那个虚拟寄存器 (对应 ebp) 值 0x0019FF80，存放到 0019FF4C。

我们再看下一条 handler，这条将偏移为 0x18 的那个虚拟寄存器 (对应 ebp) 值 0x0019FF80，存放到 0019FF4C。

我们再看下一条 handler，这条将偏移为 0x0 的那个虚拟寄存器 (对应 edi) 值 0x00401000，存放到 0019FF48。

我们再看下一条 handler，这条将偏移为 0x1C 的那个虚拟寄存器值 0xAD7B4B1E，存放到 0019FF44。

以上分析中，除了 ecx，edx，esi，edi，这四个寄存器的值刚开始都为 401000，命名的时候比较混乱以为，其他寄存器的值在虚拟寄存器上都和他们在入口点的值一样。这也是我分析到这边坚持下去的原因吧。  
这边就是把虚拟寄存器全部吐到堆栈上了，等待从堆栈弹到各个真是寄存器中。

我们再看下一条 handler，

```
004056DD    66:D3FF         sar di,cl
004056E0    66:0FBAF3 06    btr bx,0x6
004056E5    38DE            cmp dh,bl
004056E7    89EC            mov esp,ebp;执行完esp = 0019FF44
004056E9    66:11E1         adc cx,sp
004056EC    FC              cld
004056ED    59              pop ecx ;执行完ecx = AD7B4B1E  ;唯一作用就是修改此时esp的值                 
004056EE    9F              lahf
004056EF    5B              pop ebx ;执行完ebx = 00401000 ;唯一作用就是修改此时esp的值                         
004056F0    66:0FC1EE       xadd si,bp
004056F4    D5 B9           aad 0xB9
004056F6    66:0FC1D6       xadd si,dx
004056FA    5D              pop ebp;执行完ebp = 0019FF80                           
004056FB    D4 7A           aam 0x7A
004056FD    5E              pop esi;执行完esi = 00401000                        
004056FE    3F              aas
004056FF    D2D4            rcl ah,cl
00405701    E8 F3F2FFFF     call HelloASM.004049F9

```

```
004049F9    66:21DF         and di,bx
004049FC    8B7C24 04       mov edi,dword ptr ss:[esp+0x4]  ;垃圾指令
00404A00    99              cdq
00404A01    8B5C24 08       mov ebx,dword ptr ss:[esp+0x8]  ;执行完ebx = 00228000
00404A05    F5              cmc
00404A06    80FA 7A         cmp dl,0x7A
00404A09    8B7C24 0C       mov edi,dword ptr ss:[esp+0xC]  ;执行完edi = 00401000
00404A0D    80E5 E5         and ch,0xE5
00404A10    0FBAF8 0A       btc eax,0xA
00404A14    66:0FB6C1       movzx ax,cl
00404A18    8B4424 10       mov eax,dword ptr ss:[esp+0x10]  ;执行完eax =  0019FFCC
00404A1C    66:81D1 6348    adc cx,0x4863
00404A21    8B5424 14       mov edx,dword ptr ss:[esp+0x14]  ;执行完edx = 00401000
00404A25    F5              cmc
00404A26    9C              pushfd
00404A27    8B4C24 1C       mov ecx,dword ptr ss:[esp+0x1C] ;执行完ecx = 00401000
00404A2B    9C              pushfd
00404A2C    F5              cmc
00404A2D    FF7424 24       push dword ptr ss:[esp+0x24]    
00404A31    9D              popfd
00404A32    66:C74424 24 3E>mov word ptr ss:[esp+0x24],0xAA3E
00404A39    886424 0C       mov byte ptr ss:[esp+0xC],ah
00404A3D    FF7424 28       push dword ptr ss:[esp+0x28]
00404A41    C2 2C00         retn 0x2C;返回到 401005

```

最后我们给堆栈标一下对应寄存器的位置。

```
0019FF4C   0019FF80  ebp
0019FF50   00401000  esi
0019FF54   461BFFD3
0019FF58   002A6000  ebx
0019FF5C   00401000  edi
0019FF60   0019FFCC  eax
0019FF64   00401000  edx
0019FF68   00401000  ecx
0019FF6C   00000246  eflag

```

总结，2.12.3 版本和 1.6 版本相比确实困难了不少，比较有意思的是在一个流程块里面虚拟寄存器，就已经进行了位置互换，我分析 1.6 之前的只在每个流程块里不一样。还有就是分析到中间的那个修改 esi 的值用了加密，而最后离开虚拟机的，那个返回值却连加密都没有，直接放在伪代码上面，在分析的 1.6 版本之前都是有加密的，还有修改和读取虚拟寄存器对应的伪代码，本来只要一个字节就行，现在还拆分成了两个字节，等等。这些的改变，使编写一款分析插件相较于之前的版本难度和工作量增加了不少，但是总体的运行框架没有变，这也是我可以根据分析之前的版本，预测出每一步都是干什么，本来还想去分析它的反调试整个执行流程，但是之前 1.6 版本已经分析过了，我想再分析，可能看不出什么新意吧，那就显得很枯燥，没意思了，那下一篇分析就直接进入 3.0 时代了，想想都觉得兴奋。

[[培训] 内核驱动高级班，冲击 BAT 一流互联网大厂工作，每周日 13:00-18:00 直播授课](https://www.kanxue.com/book-section_list-173.htm)

最后于 5 天前 被阿强编辑 ，原因：

[#VM 保护](forum-4-1-4.htm) [#调试逆向](forum-4-1-1.htm)

上传的附件：

*   [vmp.part1.rar](javascript:void(0);) （5.00MB，15 次下载）
*   [vmp.part2.rar](javascript:void(0);) （5.00MB，15 次下载）
*   [vmp.part3.rar](javascript:void(0);) （3.91MB，17 次下载）