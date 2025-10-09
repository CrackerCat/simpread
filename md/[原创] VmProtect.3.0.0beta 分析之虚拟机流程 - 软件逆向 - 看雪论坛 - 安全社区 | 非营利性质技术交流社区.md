> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-288656.htm)

> [原创] VmProtect.3.0.0beta 分析之虚拟机流程

终于可以开始分析 VmProtect3.x，以上的版本了，从刚开始接触 vmp，那时是从 0.7 开始的，算是最早的版本了吧，那时候对虚拟机，一点概念都没有，什么虚拟寄存器，什么伪代码，为什么虚拟机里面没有寄存器的概念等等，慢慢的从了解其基本执行流程，到了解其混淆，一步步的学习，到开始分析 3.x 版本，还是蛮有成就感的，又可以接触新的东西了，而且是很多，从 3.x 版本开始，代码进行了重构，编写语言改用了 C++，界面用上了 QT 变得更好看了，加密后的文件的大小也越来越大，也是因为计算机性能的大幅度提升，不用再考虑太多的性能方面的东西，上一篇文章，我讲的是 2.12.3 版本的，加密前是 3kb，加密后才 12kb，然后这篇的例子，由于这个版本的主程序，还是有点问题的，我在 win10 运行不了，于是我放 xp 虚拟机上跑，给程序加壳的时候也是状况百出，但是经过一顿测试，拿了一堆程序，又改来改去加密的代码，终于弄出一个较为满意的例子。然后这个程序加壳前，大小 179kb，加壳后 590kb。大小直接增加 400 多 kb，秉承着，柿子专挑乱的捏，选择了 VmProtect.3.0.0beta，这个 beta 是测试版的意思。  
这次给大家的附件没有虚拟机主程序，你们去论坛下载就可以了，然后我说下，设置啥的，首先加密的是入口点后面的的下面两行代码，

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

也是吸取上一篇文章的教训了，给六个寄存器都赋值，加密的是 0040A5C6 和 0040A5CB 这两条代码，然后选项那边，把是都改成否，英文版的，yes 全改 no，最后 Compilation Type(编译类型)，这边选择 Virtualization（虚拟化）。我们把编译成功的 123.vmp.exe 拖到 OD 上

```
0040A5C6   .- E9 CCC20800   jmp 123_vmp.00496897
0040A5CB   >^ FFE7          jmp edi                     
0040A5CD   .  57            push edi                    
0040A5CE   .  C3            retn
0040A5CF   .  4F            dec edi                     
0040A5D0   .  C3            retn
0040A5D1      90            nop

```

0040A5C6 这边的代码被替换成了 jmp 指令，0040A5CB 这边的代码可能是被替换成某个 handler 了吧。我们接着往下单步

```
00496897    68 5AFA2B26     push 0x262BFA5A
0049689C    E8 05BFFBFF     call 123_vmp.004527A6

```

到这里还是和我们前面分析的 vmp1.6 一样，push，call 组合。我们接着往下单步

```
004527A6    9C              pushfd
------省略------
004527CD    8B7424 28       mov esi,dword ptr ss:[esp+0x28]
------省略------
00452811    8BEC            mov ebp,esp
------省略------
0045281C    8DA424 40FFFFFF lea esp,dword ptr ss:[esp-0xC0]
------省略------
0045282B   >  8BDE           mov ebx,esi;ebx存放esi的值作为密钥   
------省略------
0045283B    8D3D 3B284500   lea edi,dword ptr ds:[0x45283B]
00452841    8B06            mov eax,dword ptr ds:[esi]
00452843    F9              stc
00452844    8DB6 04000000   lea esi,dword ptr ds:[esi+0x4]
0045284A    33C3            xor eax,ebx;解密
0045284C    48              dec eax解密
0045284D    66:3D 8A58      cmp ax,0x588A
00452851    66:85D7         test di,dx
00452854    0FC8            bswap eax解密
00452856    2D 6D7E3B00     sub eax,0x3B7E6D解密
0045285B    35 412CD300     xor eax,0xD32C41解密
00452860    F6C6 C6         test dh,0xC6
00452863    33D8            xor ebx,eax;生成下一次解密的密钥
00452865    F8              clc
00452866    03F8            add edi,eax ;算出handler的地址
00452868    57              push edi
00452869    C3              retn ;执行handler

```

这边因为虚拟机大小和之前变得大很多，我贴代码不能像以前一样全部贴出来，太占用篇幅了，关键的留住，其他的用省略表示，然后这一大段代码，我们分析的时候可以翻开之前我分析的 1.64 版本的那个例子，我也放在附件里面了，这边我们执行到 004527CD 的时候可以看到，这里和 1.64 版本的区别是少了那个重复的寄存器。004527CD 这边的代码执行完，这时候这个 esi 是加密了的，00452811 这条代码执行完 esi，才解密出来，前面的那些对其他寄存器的操作统统都是垃圾指令我们只看 esi 就行，00452811 这条代码我们也可以在 1.64 版本找到他的位置，0045281C 这条代码等价于 sub esp,0xC0。0045283B 这条代码等价于 mov edi，0045283B，这边在我们前面分析过的 vmp 里面从来没见过了，这个 edi 不是 handler 的基地址，只能说用于计算 handler 的地址，我们再往下单步，00452841 这条指令是从 esi 读取 4 字节，以前分析的都是读取 1 字节，0045284A 这条指令和 00452863 的指令是成对出现的，在我以前分析过 1.2 版本伪代码解密的时候，有详细分析过，就是用伪代码作为密钥，但是执行一次伪代码就变一次，分析的时候还觉得挺复杂，那时候有三种可能，xor 对 xor，add 对 add，sub 对 sub。不一样的是这边是 4 字节，分析到这边总结一下变化，果然重构之后很强大，这里作者已经把 handler 隐藏了，这里面所有的改动就一个目的隐藏 handler，以前那种 eax*4 + 基地址 = handler 地址，类似 c++ 虚函数的时代已经过去了，esp 存放虚拟寄存的基地址，之前版本是 edi，那么现在 edi 干什么呢? 之前他作为虚拟寄存器基地址默默无闻，现在可牛逼了，现在成了存放 handler 地址的寄存器，配合 push 和 retn 去执行各个 handler，遵循一个公式，edi = 解密 (伪代码) + edi, 所以说我们要算出所有 handler 的地址，上一条执行的 handler 的地址。核心思路还是一样，只不过实现起来和以前发生了很大的改变。作者的目的就一个，怕我们像以前版本一样知道所有 handler 的地址，然后把所有伪代码都翻译成正常指令。

我们执行到 00452869 的代码接着单步，就来到了第一条 handler

```
00447687    0FB606          movzx eax,byte ptr ds:[esi];取出一个字节的伪代码
0044768A    66:D3C9         ror cx,cl
0044768D    81C6 01000000   add esi,0x1;;伪代码指针加1
00447693    03CF            add ecx,edi                 
00447695    81F1 7B41E822   xor ecx,0x22E8417B
0044769B    66:0FBAF1 41    btr cx,0x41
004476A0    32C3            xor al,bl ;解密伪代码
004476A2    0FBFCC          movsx ecx,sp
004476A5    F6D0            not al ;解密伪代码
004476A7    0f94c5          sete ch
004476AA    FEC8            dec al ;解密伪代码
004476AC    D3F9            sar ecx,cl
004476AE    0FB7CB          movzx ecx,bx
004476B1    F6D0            not al ;解密伪代码
004476B3    0FBDCE          bsr ecx,esi                 
004476B6    F6D8            neg al ;解密伪代码
004476B8    0BCF            or ecx,edi                  
004476BA    66:3BC6         cmp ax,si
004476BD    32D8            xor bl,al;生成下次解密的密钥
004476BF    c1d1 f3         rcl ecx,0xf3
004476C2    86ED            xchg ch,ch
004476C4    66:13CE         adc cx,si
004476C7    8B4C25 00       mov ecx,dword ptr ss:[ebp];取出堆栈的值（对应push 0）
004476CB    A8 8D           test al,0x8D
004476CD    81C5 04000000   add ebp,0x4;ebp指向下一个堆栈的值（对应push esi）
004476D3    66:F7C7 5744    test di,0x4457
004476D8    890C04          mov dword ptr ss:[esp+eax],ecx ;存入虚拟寄存器（对应Vm_Relocation）
004476DB    66:0FBAF8 99    btc ax,0x99
004476E0    03C7            add eax,edi                 
004476E2    8B06            mov eax,dword ptr ds:[esi];从esi读取4字节
004476E4    81C6 04000000   add esi,0x4;伪代码指针加4
004476EA    F8              clc
004476EB    33C3            xor eax,ebx ;解密伪代码
004476ED    48              dec eax  ;解密伪代码
004476EE    F8              clc
004476EF    F5              cmc
004476F0    0FC8            bswap eax;解密伪代码
004476F2    66:F7C2 FD44    test dx,0x44FD
004476F7    8D80 9381C4FF   lea eax,dword ptr ds:[eax-0x3B7E6D] ;解密伪代码
004476FD    35 412CD300     xor eax,0xD32C41;解密伪代码
00447702    33D8            xor ebx,eax;生成下次解密的密钥
00447704    85DA            test edx,ebx
00447706    03F8            add edi,eax;生成handler的地址
00447708  ^ FFE7            jmp edi;等价于push retn组合             

```

上面这条 handler 虽然和以前分析的一样，是填充虚拟寄存器用的，但是也有很大的变化，上一篇我们分析的那个 2.12.3 版本，虚拟寄存器的偏移直接在伪代码上可以看出来，但是这边采用了双重加密，关于双重加密，我出的课程[《VMProtect 虚拟机逆向入门》](https://www.kanxue.com/book-section_list-200.htm) 有对主程序的逆向分析，其中对这个双重加密有详细的分析。这边我简单说一下，这个加密方式早在我分析 1.2 版的时候就有了，首先双重加密第一次就是用譬如 not，inc，bswap 等一些算术指令的组合进行加密，第二次就是利用伪代码地址作为密钥使用 xor，add，sub 其中一个加密, 这个伪代码地址也会加密生成新的密钥，所以解密的时候正好反过来，先用伪代码地址作为密钥使用 xor，add，sub 其中一个解密，接着再用譬如 not，inc，bswap 等一些算术指令的组合进行解密，这才解密完成，然后就是再算出新的密钥。大概就是这个意思了，然后这个伪代码地址加密的长度可以是 4 字节 2 字节和 1 字节，如果是 2 个字节和 1 个字节那就更新到 ebx 对应的 bx 或 bl 位置，这个解密伪代码地址用 ebx 存放，第一次加密或者解密的时候就 esi 的值。其实这个我们大致了解一下就可以了，反正都是加密了，这边还有很多无关紧要的垃圾指令，其实我们了解他的行为之后，那些垃圾指令一眼就看出来了。前两篇文章我们分析的时候没有考虑到的一个问题，现在还是要考虑一下，就是用 vm_+ 寄存器名，来指代对应的虚拟寄存器，这个虚拟寄存器没什么深奥的，一句话就是用来存放真是寄存器的，仅此而已。eax 对应的虚拟寄存器就是 vm_eax, 这样子我打字起来不会那么费劲，004476D8 这句代码的注释，Vm_Relocation 就是指存放重定位的虚拟寄存器了，这个重定位一直都是 0，在堆栈上占坑用的，前面我们说过 esp 存放是虚拟寄存器基地址的，004476D8 上的这条指令看到了吧，填充好对应的虚拟寄存器之后，就又去伪代码上读取 4 字节值，解密后与这条 handler 的地址相加计算出下一条 handler 的地址了，然后去执行。这与我们以前分析的 handler 区别就是以前执行完，跳回一个固定的地址解密出下一条 handler 地址去执行，不断的循环，而这里的每一条 handler 由两部分构成，第一部分实现 handler 的功能，第二部分，计算下一条 handler 地址并执行。我感觉为啥，文件会突然间变得那么大，就好比以前电脑是稀罕物的时候大家没电脑的时候去网吧上网，全世界的所有电脑总和就好比文件大小，现在普及了，几乎每个人都买的起电脑，那么全世界电脑总数就突然间变得非常大了，一个道理。

我们接着看下一条 handler，同样是填充虚拟寄存的 handler，虽然不是同一条 handler 但是和上一条功能一样，这里把 esi 的值填充到偏移为 0x30 的虚拟寄存器中，那么这个虚拟寄存器就命名为 vm_esi。上面一个 handler 忘记记录了，这边记录一下是偏移为 0x2c 的虚拟寄存器，填充重定位值 0，那么偏移为 0x2c 的虚拟寄存器记为 vm_relocation。

我们接着看下一条 handler，这里把 ebp 的值填充到偏移为 0x24 的虚拟寄存器中，那么这个虚拟寄存器就命名为 vm_ebp。

我们接着看下一条 handler，这里把 ebx 的值填充到偏移为 0x4 的虚拟寄存器中，那么这个虚拟寄存器就命名为 vm_ebx。

我们接着看下一条 handler，这里把 eax 的值填充到偏移为 0x34 的虚拟寄存器中，那么这个虚拟寄存器就命名为 vm_eax。

我们接着看下一条 handler，这里把 edi 的值填充到偏移为 0x18 的虚拟寄存器中，那么这个虚拟寄存器就命名为 vm_edi。

我们接着看下一条 handler，这里把 ecx 的值填充到偏移为 0x0 的虚拟寄存器中，那么这个虚拟寄存器就命名为 vm_ecx。

我们接着看下一条 handler，这里把 edx 的值填充到偏移为 0x1c 的虚拟寄存器中，那么这个虚拟寄存器就命名为 vm_edx。

我们接着看下一条 handler，这里把 eflags 的值填充到偏移为 0x3c 的虚拟寄存器中，那么这个虚拟寄存器就命名为 vm_eflags。

我们接着看下一条 handler，这里把地址为 0x49689c 的 call 的返回值填充到偏移为 0x20 的虚拟寄存器中，那么这个虚拟寄存器就命名为 vm_retaddr。

我们接着看下一条 handler，这里把地址为 0x496897 的 push 的值填充到偏移为 0x28 的虚拟寄存器中，那么这个虚拟寄存器就命名为 vm_push。

到这里我们制作一个虚拟寄存器分布表, 这个虚拟寄存器总的大小是 0x40，和我之前分析的文章所讲的版本一样，我分析 1.2 版本的时候就这样，可以存放 16 个寄存器，每个大小 4 字节，但是真正需要的不用这么多，所以多出来的可以用来计算，和用来搞混淆。  
0019FE88 vm_ecx vm_ebx 空闲 空闲  
0019FE98 空闲 空闲 vm_edi vm_edx  
0019FEA8 vm_retaddr vm_ebp vm_push vm_relocation  
0019FEB8 vm_esi vm_eax 空闲 vm_eflags

我们接着看下一条 handler，这里从伪代码取出 4 字节数，然后解密出地址 00415035，然后存放到 [0019FF70] 里面，接着又从伪代码中取出 4 字节数，解密出下一条 handler 地址。

```
004840A4    03F8            add edi,eax ;算出下一条handler地址
004840A6  ^ E9 1319FCFF     jmp 123_vmp.004459BE;这条代码不是push edi，或者jmp edi。

```

```
004459BE    8D4424 60       lea eax,dword ptr ss:[esp+0x60]
004459C2    3BE8            cmp ebp,eax
004459C4  ^ 0F87 89AFFFFF   ja 123_vmp.00440953 ;混淆，永远都是跳走

```

```
00440953  ^\FFE7            jmp edi                            

```

上面这个代码，在我讲上一篇文章混淆的时候提到过，就是固定跳转的。

我们接着看下一条 handler，我们对照上面的虚拟寄存器分布表很容易看出来是，取出 vm_eflags 的值，然后存放到  
[0019FF6C] 里面。

我们接着看下一条 handler，这条 handler 是取出 vm_edx 的值，然后存放到 [0019FF68] 里面。

我们接着看下一条 handler，这条 handler 是从伪代码读取 4 字节的值，然后解密出来的值是 0x12345678, 这个值有印象吧，然后存放到 [0019FF64] 里面。

我们接着看下一条 handler，这里把 0x12345678 填充到偏移为 0x14 的空闲虚拟寄存器中。

我们接着看下一条 handler，这条 handler 是取出 vm_ecx 的值，然后存放到 [0019FF64] 里面。

我们接着看下一条 handler，这条 handler 是取出 vm_edi 的值，然后存放到 [0019FF60] 里面。

我们接着看下一条 handler，这条 handler 是取出偏移为 0x14 的虚拟寄存器的值也就是 0x12345678，然后存放到 [0019FF5c] 里面。

我们接着看下一条 handler，这条 handler 是取出 vm_ebx 的值，然后存放到 [0019FF58] 里面。

我们接着看下一条 handler，这条 handler 是取出 vm_ebp 的值，然后存放到 [0019FF54] 里面。

我们接着看下一条 handler，这条 handler 是取出 vm_esi 的值，然后存放到 [0019FF50] 里面。

上面这几个 handler 都是将虚拟寄存器弹出到堆栈，这里我们再制作一个表格，标出堆栈上虚拟寄存器的分布，

```
0019FF50   51515151 ;vm_esi
0019FF54   0019FF80 ;vm_ebp
0019FF58   BBBBBBBB ;vm_ebx
0019FF5C   12345678 ;0x12345678
0019FF60   D1D1D1D1 ;vm_edi
0019FF64   CCCCCCCC ;vm_ecx
0019FF68   DDDDDDDD ;vm_edx
0019FF6C   00000246 ;vm_eflags
0019FF70   00415035 ;00415035

```

我们上面的这个表，好像没有 vm_eax, 为啥呢, 这里就是因为我们虚拟化了 mov eax，12345678 这条指令的缘故了，其实 0019FF5C 就是 vm_eax, 另外 0019FF70 的值 00415035 就是我们虚拟化 jmp 00415035 的结果了。我们看下最后一条 handler，

```
0047F585    8BE5            mov esp,ebp ;修改esp，配合后面的出栈指令
0047F587    0FBFF6          movsx esi,si
0047F58A    5E              pop esi     ;恢复esi                             
0047F58B    9F              lahf
0047F58C    0FBFDA          movsx ebx,dx
0047F58F    F7C5 A23F0045   test ebp,0x45003FA2
0047F595    5D              pop ebp     ;恢复ebp   
0047F596    5B              pop ebx     ;恢复ebx    
0047F597    98              cwde
0047F598    66:0FBAFA C4    btc dx,0xC4
0047F59D    66:98           cbw
0047F59F    58              pop eax     ;恢复eax  
0047F5A0    c0f9 28         sar cl,0x28
0047F5A3    5F              pop edi     ;恢复edi                                 
0047F5A4    66:0FB6D0       movzx dx,al
0047F5A8    59              pop ecx     ;恢复ecx   
0047F5A9    5A              pop edx     ;恢复edx  
0047F5AA    F6C3 95         test bl,0x95
0047F5AD    9D              popfd       ;恢复eflags 
0047F5AE    C3              retn        ;相当于jmp 00415035

```

上面这个代码中 执行到 0047F59F 上的指令时，此时堆栈上的值就是 12345678，执行完 eax = 12345678，就是模拟了 mov eax，12345678，然后执行到 0047F59F 上的指令时，此时堆栈上的值就是 00415035，就是模拟了 jmp 00415035。我没有虚拟化其他比较复杂的指令，原理都是差不多的，到此整个虚拟机执行流程就完毕了，这样看的起来还蛮简单的，文件看上去挺大，可是执行的代码并不多，除了新的东西，感觉还没分析 2.12.3 版本复杂。毕竟这是测试版，主要是实现功能而已，混淆的并不多，我还以为会分析的很久，没想到这么快就完成了，不过也看到很多新的东西，还是收获满满。

[[培训] 内核驱动高级班，冲击 BAT 一流互联网大厂工作，每周日 13:00-18:00 直播授课](https://www.kanxue.com/book-section_list-173.htm)

最后于 5 天前 被阿强编辑 ，原因：

[#调试逆向](forum-4-1-1.htm) [#VM 保护](forum-4-1-4.htm)

上传的附件：

*   [vmp.rar](javascript:void(0);) （394.93kb，15 次下载）