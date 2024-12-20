> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-284883.htm)

> [原创]vm 虚拟机的初探

本文仅限于技术讨论，不得用于非法途径，后果自负。

这是萌新第一次逆向 vmp，主要是分享分析过程，这里借鉴了金罡大佬的文章 [https://bbs.kanxue.com/thread-282300.htm，vmp 的内存布局和 context](https://bbs.kanxue.com/thread-282300.htm%EF%BC%8Cvmp%E7%9A%84%E5%86%85%E5%AD%98%E5%B8%83%E5%B1%80%E5%92%8Ccontext) 在我选用的这个版本差别并不大。选用的 TT 中 libEncryptor.so 中被 vm 的 jnionload。

用 ida 打开 so 定位到 AF8 ，看一下 cfg  
![](https://bbs.kanxue.com/upload/attach/202412/856431_Z7A2UDPAZ223QPU.webp)  
下面这一排就是 handle 的实现了  
![](https://bbs.kanxue.com/upload/attach/202412/856431_WTS3BWA9NZCXBMM.webp)  
不过 handle 太多了 我只会还原被执行的指令集。

从下面两张图可以看出 vm 指令集 4 字节对齐和 arm 一致 ，在后面的逆向中发现开发者并没有实现指令集或者说指令集就是 arm64  
![](https://bbs.kanxue.com/upload/attach/202412/856431_VBMHNQFYPHJHMSU.webp)  
![](https://bbs.kanxue.com/upload/attach/202412/856431_XKSPETE27PU83D5.webp)  
还原指令的第一步就是解析出指令中携带的信息，一般来说，有这几个信息  
1. 指令类型  
2. 寄存器信息  
3. 立即数  
可以看到在虚拟机循环的开头就是虚拟机取出指令各个信息的过程  
![](https://bbs.kanxue.com/upload/attach/202412/856431_8NHXECEN27NUMVN.webp)

可以看到 指令类型 sourceREG destREG opcode 这几个关键的信息

可以理解为虚拟机使用的寄存器 堆栈 等信息  
这里可以参考金罡大佬的文章 [https://bbs.kanxue.com/thread-282300.htm](https://bbs.kanxue.com/thread-282300.htm) 这个版本没什么变化，唯一注意的是我并不是通过汇编而是 f5 代码来确定 coutext 的结构的，而开发者为了对抗，这个 context 结构体是负指针寻址的 所以需要自己 patch 代码  
比如 STP X3, X20, [X19,#-0x100]=>STP X3, X20, [X19,#0x100]  
这个方法中只需要 patch 所有有关 x19 的指针操作即可  
效果图  
![](https://bbs.kanxue.com/upload/attach/202412/856431_UQ4GSB3966YM4MG.webp)  
![](https://bbs.kanxue.com/upload/attach/202412/856431_F6BV8VEMQAH5DT6.webp)

这个可能是最头疼的一点，明明开发者写的就是一个 while switch 可是被编译器优化后代码复用，各种跳转比如  
![](https://bbs.kanxue.com/upload/attach/202412/856431_STA3C36W7SNDPXQ.webp)  
可能你找 handle 的时间都比逆向 handle 的时间长  
这里，我使用了一个实验性质的方法  
利用 trace 在剔除控制流指令后重新 patch  
效果图  
![](https://bbs.kanxue.com/upload/attach/202412/856431_8A485P9KPWN4VJE.webp)  
可以看到 所有 handle 都被打印出来了  
指令缩短 变成单次打印单个 handle  
![](https://bbs.kanxue.com/upload/attach/202412/856431_38A3KMQAPUW9HB5.webp)  
这样我觉得是提高了我的分析效率

这个是我觉得最繁琐的地方，我看了我还原出来的指令和金罡大佬的还原，还是细节不够  
![](https://bbs.kanxue.com/upload/attach/202412/856431_KRCBZ3QS3KHUYK6.webp)  
![](https://bbs.kanxue.com/upload/attach/202412/856431_EX47VYQ4FVEGNP3.webp)  
对比可以看到 mov 少了个 z，这就是细节不够，另外还有对 arm 指令集不够熟悉一些偏门指令如果不知道那就完犊子了

这是我还原最后一个 handle 看出来的 原先我以为是 cmp 但是不对劲，简而言之就是每一次 call 后会检查返回值 如果返回值对 vm_pc+2 结合 br 指令集 lr = pc+2 刚好四字节对齐，如果不对 哦吼 pc +2 指令集对齐直接挂 随机崩溃  
![](https://bbs.kanxue.com/upload/attach/202412/856431_HR6FQWMAR8Q4GPB.webp)  
![](https://bbs.kanxue.com/upload/attach/202412/856431_2PPFZD7TTYSKMKX.webp)  
br handle  
![](https://bbs.kanxue.com/upload/attach/202412/856431_AJMK62ZV3D2X47M.webp)

`/``/` `unsigned` `int` `code` `/``/` `Bit_28_State` `=` `code &` `0x10000000``;` `/``/` `Bit_29_State` `=` `code &` `0x20000000``;` `/``/` `Bit_30_State` `=` `code &` `0x40000000``;` `/``/` `v17` `=` `(code >>` `11``) &` `2` `| (code >>` `31``) | (code >>` `11``) &` `4` `| (code >>` `11``) &` `8` `| (code >>` `11``) &` `0x10``;``/``/` `提取第``31``位 提取第``12``,``13``,``14``,``15``位 组合` `/``/`                                           `/``/` `[``15``,``14``,``13``,``12``,``31``]` `/``/` `sourceREG` `=` `(code >>` `21``) &` `0x1F``;` `/``/` `取第``22` `=``>``26``的值` `/``/`                                           `/``/` `[``26``,``25``,``24``,``23``,``22``]` `/``/` `destREG` `=` `HIWORD(code) &` `0x1F``;` `/``/` `取``17` `>` `21``位` `/``/`                                           `/``/` `[``21``,``20``,``19``,``18``,``17``]` `/``/` `Bit_31_State` `=` `code &` `0x80000000``;` `/``/` `Bit_30_State_1` `=` `code &` `0x4000000``;` `/``/` `Bit_31_State_1` `=` `code &` `0x8000000``;` `var` `type` `=` `code.``and``(``0x3f``)` `var code` `=` `this.vm_handles` `var sourceREG` `=` `code.shr(``21``).``and``(``0x1F``)` `var destREG` `=` `code.shr(``16``).``and``(``0x1F``)`   `var Bit_28_State` `=` `code.``and``(``0x10000000``)` `var Bit_29_State` `=` `code.``and``(``0x20000000``)` `var Bit_30_State` `=` `code.``and``(``0x40000000``)` `var Bit_31_State` `=` `code.``and``(``0x80000000``)` `var Bit_30_State_1` `=` `code.``and``(``0x4000000``)` `var Bit_31_State_1` `=` `code.``and``(``0x8000000``)` `var intv17` `=` `code.toUInt32()` `var v17` `=` `(intv17 >>>` `11``) &` `2` `| (intv17 >>>` `31``) | (intv17 >>>` `11``) &` `4` `| (intv17 >>>` `11``) &` `8` `| (intv17 >>>` `11``) &` `0x10` `let opcode` `=` `code.``and``(``0xF000``).``or``(Bit_30_State_1.shr(``20``)).``or``(code.shr(``6``).``and``(``0x3F``)).``or``(Bit_31_State_1.shr(``20``)).``or``(Bit_28_State.shr(``20``)).``or``(Bit_29_State.shr(``20``)).``or``(Bit_30_State.shr(``20``)).``or``(Bit_31_State.shr(``20``))` `/``/` `unsigned` `int` `code`

`/``/` `Bit_28_State` `=` `code &` `0x10000000``;`

`/``/` `Bit_29_State` `=` `code &` `0x20000000``;`

`/``/` `Bit_30_State` `=` `code &` `0x40000000``;`

`/``/` `v17` `=` `(code >>` `11``) &` `2` `| (code >>` `31``) | (code >>` `11``) &` `4` `| (code >>` `11``) &` `8` `| (code >>` `11``) &` `0x10``;``/``/` `提取第``31``位 提取第``12``,``13``,``14``,``15``位 组合`

`/``/`                                           `/``/` `[``15``,``14``,``13``,``12``,``31``]`

`/``/` `sourceREG` `=` `(code >>` `21``) &` `0x1F``;` `/``/` `取第``22` `=``>``26``的值`

`/``/`                                           `/``/` `[``26``,``25``,``24``,``23``,``22``]`

`/``/` `destREG` `=` `HIWORD(code) &` `0x1F``;` `/``/` `取``17` `>` `21``位`

`/``/`                                           `/``/` `[``21``,``20``,``19``,``18``,``17``]`

`/``/` `Bit_31_State` `=` `code &` `0x80000000``;`

`/``/` `Bit_30_State_1` `=` `code &` `0x4000000``;`

`/``/` `Bit_31_State_1` `=` `code &` `0x8000000``;`

`var` `type` `=` `code.``and``(``0x3f``)`

`var code` `=` `this.vm_handles`

`var sourceREG` `=` `code.shr(``21``).``and``(``0x1F``)`

`var destREG` `=` `code.shr(``16``).``and``(``0x1F``)`

 

`var Bit_28_State` `=` `code.``and``(``0x10000000``)`

`var Bit_29_State` `=` `code.``and``(``0x20000000``)`

`var Bit_30_State` `=` `code.``and``(``0x40000000``)`

`var Bit_31_State` `=` `code.``and``(``0x80000000``)`

`var Bit_30_State_1` `=` `code.``and``(``0x4000000``)`

`var Bit_31_State_1` `=` `code.``and``(``0x8000000``)`

`var intv17` `=` `code.toUInt32()`

`var v17` `=` `(intv17 >>>` `11``) &` `2` `| (intv17 >>>` `31``) | (intv17 >>>` `11``) &` `4` `| (intv17 >>>` `11``) &` `8` `| (intv17 >>>` `11``) &` `0x10`

`let opcode` `=` `code.``and``(``0xF000``).``or``(Bit_30_State_1.shr(``20``)).``or``(code.shr(``6``).``and``(``0x3F``)).``or``(Bit_31_State_1.shr(``20``)).``or``(Bit_28_State.shr(``20``)).``or``(Bit_29_State.shr(``20``)).``or``(Bit_30_State.shr(``20``)).``or``(Bit_31_State.shr(``20``))` `decode(): string {` `/``/``ArrayBuffer to ptr` `var` `type` `=` `this.vm_handles.``and``(``0x3F``)` `/``/` `unsigned` `int` `code` `/``/` `Bit_28_State` `=` `code &` `0x10000000``;` `/``/` `Bit_29_State` `=` `code &` `0x20000000``;` `/``/` `Bit_30_State` `=` `code &` `0x40000000``;` `/``/` `v17` `=` `(code >>` `11``) &` `2` `| (code >>` `31``) | (code >>` `11``) &` `4` `| (code >>` `11``) &` `8` `| (code >>` `11``) &` `0x10``;``/``/` `提取第``31``位 提取第``12``,``13``,``14``,``15``位 组合` `/``/`                                           `/``/` `[``15``,``14``,``13``,``12``,``31``]` `/``/` `sourceREG` `=` `(code >>` `21``) &` `0x1F``;` `/``/` `取第``22` `=``>``26``的值` `/``/`                                           `/``/` `[``26``,``25``,``24``,``23``,``22``]` `/``/` `destREG` `=` `HIWORD(code) &` `0x1F``;` `/``/` `取``17` `>` `21``位` `/``/`                                           `/``/` `[``21``,``20``,``19``,``18``,``17``]` `/``/` `Bit_31_State` `=` `code &` `0x80000000``;` `/``/` `Bit_30_State_1` `=` `code &` `0x4000000``;` `/``/` `Bit_31_State_1` `=` `code &` `0x8000000``;` `var code` `=` `this.vm_handles` `var sourceREG` `=` `code.shr(``21``).``and``(``0x1F``)` `var destREG` `=` `code.shr(``16``).``and``(``0x1F``)`   `var Bit_28_State` `=` `code.``and``(``0x10000000``)` `var Bit_29_State` `=` `code.``and``(``0x20000000``)` `var Bit_30_State` `=` `code.``and``(``0x40000000``)` `var Bit_31_State` `=` `code.``and``(``0x80000000``)` `var Bit_30_State_1` `=` `code.``and``(``0x4000000``)` `var Bit_31_State_1` `=` `code.``and``(``0x8000000``)` `var intv17` `=` `code.toUInt32()` `var v17` `=` `(intv17 >>>` `11``) &` `2` `| (intv17 >>>` `31``) | (intv17 >>>` `11``) &` `4` `| (intv17 >>>` `11``) &` `8` `| (intv17 >>>` `11``) &` `0x10` `let opcode` `=` `code.``and``(``0xF000``).``or``(Bit_30_State_1.shr(``20``)).``or``(code.shr(``6``).``and``(``0x3F``)).``or``(Bit_31_State_1.shr(``20``)).``or``(Bit_28_State.shr(``20``)).``or``(Bit_29_State.shr(``20``)).``or``(Bit_30_State.shr(``20``)).``or``(Bit_31_State.shr(``20``))`   `switch (``type``.toUInt32()) {` `case` `3``:` `case` `5``:` `case` `0xF``:` `case` `0x1A``:` `case` `0x2C``:` `case` `0x2D``:` `case` `0x3A``:` `case` `0x3F``:` `switch (``type``.toUInt32()) {` `case` `5``:` `case` `0xF``:` `case` `0x2C``:` `case` `0x3A``:` `return` `"cmp "` `+` `this.reg(sourceREG)` `+` `","` `+` `this.reg(destREG)` `}`   `case` `0x2``:` `/``/` `vm_regs[(HIWORD(vm_code_1) &` `0x1F``)` `+` `1``]` `=` `*``(_QWORD` `*``)(vm_regs[((vm_code_1 >>` `21``) &` `0x1F``)` `+` `1``]` `/``/` `+` `(__int16)(vm_code_1 &` `0xF000` `| ((vm_code_1 &` `0x4000000``) >>` `20``) | (vm_code_1 >>` `6``) &` `0x3F` `| ((vm_code_1 &` `0x8000000``) >>` `20``) | ((vm_code_1 &` `0x10000000``) >>` `20``) | ((vm_code_1 &` `0x20000000``) >>` `20``) | ((vm_code_1 &` `0x40000000``) >>` `20``) | ((vm_code_1 &` `0x80000000``) >>` `20``)));` `let opcode1` `=` `code.``and``(``0xF000``).``or``(Bit_30_State_1.shr(``20``)).``or``(code.shr(``6``).``and``(``0x3F``)).``or``(Bit_31_State_1.shr(``20``)).``or``(Bit_28_State.shr(``20``)).``or``(Bit_29_State.shr(``20``)).``or``(Bit_30_State.shr(``20``)).``or``(Bit_31_State.shr(``20``))` `return` `"ldr "` `+` `this.reg(destREG)` `+` `",["` `+` `this.reg(sourceREG)` `+` `",#"` `+` `opcode1` `+` `"]"` `case` `0xA``:` `case` `0xB``:` `case` `0xE``:` `case` `0x14``:` `case` `0x16``:` `case` `0x24``:` `case` `0x36``:` `case` `0x3b``:` `/``/` `(__int16)(code &` `0xF000` `| (Bit_30_State_1 >>` `20``) | (code >>` `6``) &` `0x3F` `| (Bit_31_State_1 >>` `20``) | (Bit_28_State >>` `20``) | (Bit_29_State >>` `20``) | (Bit_30_State >>` `20``) | (Bit_31_State >>` `20``));` `let opcode` `=` `code.``and``(``0xF000``).``or``(Bit_30_State_1.shr(``20``)).``or``(code.shr(``6``).``and``(``0x3F``)).``or``(Bit_31_State_1.shr(``20``)).``or``(Bit_28_State.shr(``20``)).``or``(Bit_29_State.shr(``20``)).``or``(Bit_30_State.shr(``20``)).``or``(Bit_31_State.shr(``20``))` `return` `"stp "` `+` `this.reg(destREG)` `+` `" ,["` `+` `this.reg(sourceREG)` `+` `","` `+` `opcode` `+` `"]"` `case` `4``:` `case` `0x20``:` `case` `0x38``:` `case` `0x3e``:` `/``/` `v39` `=` `vm_code_1 &` `0x3F``;` `/``/` `if` `(v39 >` `0x37``) {` `/``/`     `if` `(v39` `=``=` `56``) {` `/``/` `*` `(& v5` `-``> REGS` `+` `(unsigned` `int``)v19)` `=` `*` `(& v5` `-``> REGS` `+` `(unsigned` `int``)v18) ^ (vm_code_1 &` `0xF000` `| (v21 >>` `20``) | (vm_code_1 >>` `6``) &` `0x3F` `| (v22 >>` `20``) | (v14 >>` `20``) | (v15 >>` `20``) | (v16 >>` `20``) | (v20 >>` `20``));` `/``/`     `}` `/``/`     `else` `if` `(v39` `=``=` `62``) {` `/``/` `*` `(& v5` `-``> REGS` `+` `(unsigned` `int``)v19)` `=` `*` `(& v5` `-``> REGS` `+` `(unsigned` `int``)v18) | vm_code_1 &` `0xF000` `| (v21 >>` `20``) | (vm_code_1 >>` `6``) &` `0x3F` `| (v22 >>` `20``) | (v14 >>` `20``) | (v15 >>` `20``) | (v16 >>` `20``) | (v20 >>` `20``);` `/``/`     `}` `/``/` `}` `/``/` `else` `{` `/``/`     `if` `(v39` `=``=` `4``) {` `/``/`         `LODWORD(v18)` `=` `(vm_code_1 &` `0xF000` `| (v21 >>` `20``) | (vm_code_1 >>` `6``) &` `0x3F` `| (v22 >>` `20``) | (v14 >>` `20``) | (v15 >>` `20``) | (v16 >>` `20``) | (v20 >>` `20``)) <<` `16``;` `/``/` `v18` `=` `(``int``)v18;` `/``/` `v25` `=` `&vm_regs[(unsigned` `int``)v19];` `/``/` `v25[``1``]` `=` `v18;` `/``/`     `}` `/``/`     `if` `(v39` `=``=` `32``)` `/``/` `*` `(& v5` `-``> REGS` `+` `(unsigned` `int``)v19)` `=` `*` `(& v5` `-``> REGS` `+` `(unsigned` `int``)v18) & (vm_code_1 &` `0xF000` `| (v21 >>` `20``) | (vm_code_1 >>` `6``) &` `0x3F` `| (v22 >>` `20``) | (v14 >>` `20``) | (v15 >>` `20``) | (v16 >>` `20``) | (v20 >>` `20``));` `/``/` `}` `var v39` `=` `code.``and``(``0x3F``)` `let opcode2` `=` `code.``and``(``0xF000``).``or``(Bit_30_State_1.shr(``20``)).``or``(code.shr(``6``).``and``(``0x3F``)).``or``(Bit_31_State_1.shr(``20``)).``or``(Bit_28_State.shr(``20``)).``or``(Bit_29_State.shr(``20``)).``or``(Bit_30_State.shr(``20``)).``or``(Bit_31_State.shr(``20``))` `if``(v39.toUInt32() >` `0x37``){` `if``(v39.toUInt32()` `=``=` `56``){` `return` `"eor "` `+` `this.reg(destREG)` `+` `","` `+` `this.reg(sourceREG)` `+` `","` `+` `opcode2` `}``else` `if``(v39.toUInt32()` `=``=` `62``){` `return` `"orr "` `+` `this.reg(destREG)` `+` `","` `+` `this.reg(sourceREG)` `+` `","` `+` `opcode2` `}` `}``else``{` `if``(v39.toUInt32()` `=``=` `4``){` `return` `"mov "` `+` `this.reg(destREG)` `+` `","` `+` `opcode2` `}``else` `if``(v39.toUInt32()` `=``=` `32``){` `return` `"and "` `+` `this.reg(destREG)` `+` `","` `+` `this.reg(sourceREG)` `+` `","` `+` `opcode2` `}` `}` `case` `0``:` `case` `2``:` `case` `8``:` `case` `0x12``:` `case` `0x1F``:` `case` `0x21``:` `case` `0x22``:` `case` `0x27``:` `case` `0x2B``:` `case` `0x30``:` `case` `0x31``:` `case` `0x33``:` `case` `0x37``:` `return` `"mov "` `+` `this.reg(destREG)` `+` `" , "` `+` `this.reg(sourceREG)` `case` `0xD``:` `case` `0x15``:` `case` `0x1B``:` `case` `0x28``:` `/``/` `v40` `=` `vm_code_1 &` `0x3F``;` `/``/` `v41` `=` `vm_code_1 &` `0xF000` `| (v21 >>` `20``) | (vm_code_1 >>` `6``) &` `0x3F` `| (v22 >>` `20``) | (v14 >>` `20``) | (v15 >>` `20``) | (v16 >>` `20``) | (v20 >>` `20``);` `/``/` `if` `( v40` `=``=` `13` `|| v40` `=``=` `40` `)` `/``/` `{` `/``/`   `*``(&v5``-``>REGS` `+` `(unsigned` `int``)v19)` `=` `*``(&v5``-``>REGS` `+` `(unsigned` `int``)v18)` `+` `(__int16)v41;` `/``/` `}` `/``/` `else` `if` `( v40` `=``=` `21` `)` `/``/`     `{` `/``/`       `*``(&v5``-``>REGS` `+` `(unsigned` `int``)v19)` `=` `*``((``int` `*``)&v5``-``>REGS` `+` `2` `*` `v18)` `+` `(__int64)(__int16)v41;` `/``/`     `}` `let v40` `=` `code.``and``(``0x3F``)` `let v41` `=` `code.``and``(``0xF000``).``or``(Bit_30_State_1.shr(``20``)).``or``(code.shr(``6``).``and``(``0x3F``)).``or``(Bit_31_State_1.shr(``20``)).``or``(Bit_28_State.shr(``20``)).``or``(Bit_29_State.shr(``20``)).``or``(Bit_30_State.shr(``20``)).``or``(Bit_31_State.shr(``20``))` `if` `(v40.toUInt32()` `=``=` `13` `|| v40.toUInt32()` `=``=` `40``) {` `return` `"add "` `+` `this.reg(destREG)` `+` `","` `+` `this.reg(sourceREG)` `+` `","` `+` `v41` `}` `else` `if` `(v40.toUInt32()` `=``=` `21``) {` `return` `"add "` `+` `this.reg(destREG)` `+` `","` `+` `this.reg(sourceREG.shl(``1``))` `+` `","` `+` `v41` `}` `case` `0x2f``:` `/``/` `v71` `=` `vm_code_1 &` `0xFFF``;` `/``/` `HIDWORD(v72)` `=` `v71` `-` `111``;` `/``/` `LODWORD(v72)` `=` `HIDWORD(v72);` `/``/` `switch ( (unsigned` `int``)(v72 >>` `6``) )` `var v71` `=` `code.``and``(``0xFFF``)` `var v72` `=` `v71.sub(``111``)` `switch (v72.shr(``6``).toUInt32()) {` `case` `0x30``:` `if` `(v71.toInt32()` `=``=` `0xC6F``)` `/``/` `p_REGS` `=` `&v5``-``>REGS;` `/``/` `v96` `=` `*``((_DWORD` `*``)&v5``-``>REGS` `+` `2` `*` `v19) << (((vm_code_1 &` `0x10000000``) >>` `26``) &` `0xFC` `| (vm_code_1 >>` `26``) &` `3` `| ((vm_code_1 &` `0x20000000``) >>` `26``) | ((vm_code_1 &` `0x40000000``) >>` `26``));`   `return` `"ldr "` `+` `this.reg(destREG)` `+` `",["` `+` `this.reg(sourceREG.shl(``1``))` `+` `",#"` `+` `code.shr(``26``).``and``(``0xFC``).``or``(Bit_28_State.shr(``26``).``and``(``3``)).``or``(Bit_29_State.shr(``26``)).``or``(Bit_30_State.shr(``26``)).``or``(Bit_31_State.shr(``26``))` `+` `"]"` `return` `"ret"` `case` `0x15``:` `if``(v71.toInt32()` `=``=` `0x5af``)` `return` `"br "` `+` `this.reg(sourceREG)` `case` `0x19``:` `return` `"ret"` `case` `6``:` `case` `0xD``:` `case` `0x1A``:` `case` `0x20``:` `case` `0x24``:` `case` `0x26``:` `case` `0x28``:` `case` `0x33``:` `/``/`   `v73` `=` `vm_code_1 &` `0xFFF``;` `/``/`   `if` `( v73 >` `0x86E` `)` `/``/`   `{` `/``/`     `if` `( (vm_code_1 &` `0xFFF``) <``=` `0x9EE` `)` `/``/`       `goto LABEL_154;` `/``/`     `goto LABEL_64;` `/``/`   `}` `/``/`   `if` `( (vm_code_1 &` `0xFFF``) >` `0x3AE` `)` `/``/`     `goto LABEL_254;` `/``/`   `if` `( v73 !``=` `495` `)` `/``/`     `goto LABEL_218;` `/``/`   `goto LABEL_252;` `var v73` `=` `v71.toUInt32()` `if` `(v73 >` `0x86E``) {` `if` `(v71.toUInt32() <``=` `0x9EE``) {` `return` `"ret"` `}` `else` `{` `if` `( v73` `=``=` `2543` `)` `/``/` `*``(&v5``-``>REGS` `+` `v17)` `=` `*``(&v5``-``>REGS` `+` `(unsigned` `int``)v19)` `+` `*``(&v5``-``>REGS` `+` `(unsigned` `int``)v18);` `return` `"add "` `+` `this.reg(ptr(v17))` `+` `","` `+` `this.reg(destREG)` `+` `","` `+` `this.reg(sourceREG)` `}` `}` `if` `(v73 >` `0x3AE``) {` `return` `"ret"` `}` `if` `(v73 !``=` `495``) {` `return` `"ret"` `}` `else` `{` `return` `"ret"` `}` `case` `0xC``:` `case` `0x17``:` `case` `0x1F``:` `case` `0x3C``:` `/``/`             `v108` `=` `vm_code_1 &` `0xFFF``;` `/``/`   `if` `( v108 >` `0x82E` `)` `/``/`   `{` `/``/`     `if` `( v108` `=``=` `2095` `)` `/``/`     `{` `/``/`       `*``(&v5``-``>REGS` `+` `v17)` `=` `~(``*``(&v5``-``>REGS` `+` `(unsigned` `int``)v19) |` `*``(&v5``-``>REGS` `+` `(unsigned` `int``)v18));` `/``/`     `}` `/``/`     `else` `if` `( v108` `=``=` `3951` `)` `/``/`     `{` `/``/`       `*``(&v5``-``>REGS` `+` `v17)` `=` `*``(&v5``-``>REGS` `+` `(unsigned` `int``)v19) &` `*``(&v5``-``>REGS` `+` `(unsigned` `int``)v18);` `/``/`     `}` `/``/`   `}` `/``/`   `else` `if` `( v108` `=``=` `879` `)` `/``/`   `{` `/``/`     `*``(&v5``-``>REGS` `+` `v17)` `=` `*``(&v5``-``>REGS` `+` `(unsigned` `int``)v19) ^` `*``(&v5``-``>REGS` `+` `(unsigned` `int``)v18);` `/``/`   `}` `/``/`   `else` `if` `( v108` `=``=` `1583` `)` `/``/`   `{` `/``/`     `*``(&v5``-``>REGS` `+` `v17)` `=` `*``(&v5``-``>REGS` `+` `(unsigned` `int``)v19) |` `*``(&v5``-``>REGS` `+` `(unsigned` `int``)v18);` `/``/`   `}` `var v108` `=` `v71.toUInt32()` `if` `(v108 >` `0x82E``) {` `if` `(v108` `=``=` `2095``) {` `return` `"not "` `+` `this.reg(ptr(v17))` `+` `","` `+` `this.reg(destREG)` `+` `","` `+` `this.reg(sourceREG)` `}` `else` `if` `(v108` `=``=` `3951``) {` `return` `"and "` `+` `this.reg(ptr(v17))` `+` `","` `+` `this.reg(destREG)` `+` `","` `+` `this.reg(sourceREG)` `}`   `}` `else` `if` `(v108` `=``=` `879``) {` `return` `"eor "` `+` `this.reg(ptr(v17))` `+` `","` `+` `this.reg(destREG)` `+` `","` `+` `this.reg(sourceREG)` `}` `else` `if` `(v108` `=``=` `1583``) {` `return` `"orr "` `+` `this.reg(ptr(v17))` `+` `","` `+` `this.reg(destREG)` `+` `","` `+` `this.reg(sourceREG)` `}` `}` `}`   `return` `""` `}` `reg(index: NativePointer): string {` `var index1` `=` `index.toUInt32()` `if` `(index1` `=``=` `31``) {` `return` `"lr"` `}` `if` `(index1` `=``=` `29``) {` `return` `"sp"` `}` `if` `(index1` `=``=` `30``) {` `return` `"fp"` `}` `return` `"x"` `+` `index1` `}` `decode(): string {`

 `/``/``ArrayBuffer to ptr`

 `var` `type` `=` `this.vm_handles.``and``(``0x3F``)`

 `/``/` `unsigned` `int` `code`

 `/``/` `Bit_28_State` `=` `code &` `0x10000000``;`

 `/``/` `Bit_29_State` `=` `code &` `0x20000000``;`

 `/``/` `Bit_30_State` `=` `code &` `0x40000000``;`

 `/``/` `v17` `=` `(code >>` `11``) &` `2` `| (code >>` `31``) | (code >>` `11``) &` `4` `| (code >>` `11``) &` `8` `| (code >>` `11``) &` `0x10``;``/``/` `提取第``31``位 提取第``12``,``13``,``14``,``15``位 组合`

 `/``/`                                           `/``/` `[``15``,``14``,``13``,``12``,``31``]`

 `/``/` `sourceREG` `=` `(code >>` `21``) &` `0x1F``;` `/``/` `取第``22` `=``>``26``的值`

 `/``/`                                           `/``/` `[``26``,``25``,``24``,``23``,``22``]`

 `/``/` `destREG` `=` `HIWORD(code) &` `0x1F``;` `/``/` `取``17` `>` `21``位`

 `/``/`                                           `/``/` `[``21``,``20``,``19``,``18``,``17``]`

 `/``/` `Bit_31_State` `=` `code &` `0x80000000``;`

 `/``/` `Bit_30_State_1` `=` `code &` `0x4000000``;`

 `/``/` `Bit_31_State_1` `=` `code &` `0x8000000``;`

 `var code` `=` `this.vm_handles`

 `var sourceREG` `=` `code.shr(``21``).``and``(``0x1F``)`

 `var destREG` `=` `code.shr(``16``).``and``(``0x1F``)`

 

 `var Bit_28_State` `=` `code.``and``(``0x10000000``)`

 `var Bit_29_State` `=` `code.``and``(``0x20000000``)`

 `var Bit_30_State` `=` `code.``and``(``0x40000000``)`

 `var Bit_31_State` `=` `code.``and``(``0x80000000``)`

 `var Bit_30_State_1` `=` `code.``and``(``0x4000000``)`

 `var Bit_31_State_1` `=` `code.``and``(``0x8000000``)`

 `var intv17` `=` `code.toUInt32()`

 `var v17` `=` `(intv17 >>>` `11``) &` `2` `| (intv17 >>>` `31``) | (intv17 >>>` `11``) &` `4` `| (intv17 >>>` `11``) &` `8` `| (intv17 >>>` `11``) &` `0x10`

 `let opcode` `=` `code.``and``(``0xF000``).``or``(Bit_30_State_1.shr(``20``)).``or``(code.shr(``6``).``and``(``0x3F``)).``or``(Bit_31_State_1.shr(``20``)).``or``(Bit_28_State.shr(``20``)).``or``(Bit_29_State.shr(``20``)).``or``(Bit_30_State.shr(``20``)).``or``(Bit_31_State.shr(``20``))`

  

 `switch (``type``.toUInt32()) {`

 `case` `3``:`

 `case` `5``:`

 `case` `0xF``:`

 `case` `0x1A``:`

 `case` `0x2C``:`

 `case` `0x2D``:`

 `case` `0x3A``:`

 `case` `0x3F``:`

 `switch (``type``.toUInt32()) {`

 `case` `5``:`

 `case` `0xF``:`

 `case` `0x2C``:`

 `case` `0x3A``:`

 `return` `"cmp "` `+` `this.reg(sourceREG)` `+` `","` `+` `this.reg(destREG)`

 `}`

  

 `case` `0x2``:`

 `/``/` `vm_regs[(HIWORD(vm_code_1) &` `0x1F``)` `+` `1``]` `=` `*``(_QWORD` `*``)(vm_regs[((vm_code_1 >>` `21``) &` `0x1F``)` `+` `1``]`

 `/``/` `+` `(__int16)(vm_code_1 &` `0xF000` `| ((vm_code_1 &` `0x4000000``) >>` `20``) | (vm_code_1 >>` `6``) &` `0x3F` `| ((vm_code_1 &` `0x8000000``) >>` `20``) | ((vm_code_1 &` `0x10000000``) >>` `20``) | ((vm_code_1 &` `0x20000000``) >>` `20``) | ((vm_code_1 &` `0x40000000``) >>` `20``) | ((vm_code_1 &` `0x80000000``) >>` `20``)));`

 `let opcode1` `=` `code.``and``(``0xF000``).``or``(Bit_30_State_1.shr(``20``)).``or``(code.shr(``6``).``and``(``0x3F``)).``or``(Bit_31_State_1.shr(``20``)).``or``(Bit_28_State.shr(``20``)).``or``(Bit_29_State.shr(``20``)).``or``(Bit_30_State.shr(``20``)).``or``(Bit_31_State.shr(``20``))`

 `return` `"ldr "` `+` `this.reg(destREG)` `+` `",["` `+` `this.reg(sourceREG)` `+` `",#"` `+` `opcode1` `+` `"]"`

 `case` `0xA``:`

 `case` `0xB``:`

 `case` `0xE``:`

 `case` `0x14``:`

 `case` `0x16``:`

 `case` `0x24``:`

 `case` `0x36``:`

 `case` `0x3b``:`

 `/``/` `(__int16)(code &` `0xF000` `| (Bit_30_State_1 >>` `20``) | (code >>` `6``) &` `0x3F` `| (Bit_31_State_1 >>` `20``) | (Bit_28_State >>` `20``) | (Bit_29_State >>` `20``) | (Bit_30_State >>` `20``) | (Bit_31_State >>` `20``));`

 `let opcode` `=` `code.``and``(``0xF000``).``or``(Bit_30_State_1.shr(``20``)).``or``(code.shr(``6``).``and``(``0x3F``)).``or``(Bit_31_State_1.shr(``20``)).``or``(Bit_28_State.shr(``20``)).``or``(Bit_29_State.shr(``20``)).``or``(Bit_30_State.shr(``20``)).``or``(Bit_31_State.shr(``20``))`

 `return` `"stp "` `+` `this.reg(destREG)` `+` `" ,["` `+` `this.reg(sourceREG)` `+` `","` `+` `opcode` `+` `"]"`

 `case` `4``:`

 `case` `0x20``:`

 `case` `0x38``:`

 `case` `0x3e``:`

 `/``/` `v39` `=` `vm_code_1 &` `0x3F``;`

 `/``/` `if` `(v39 >` `0x37``) {`

 `/``/`     `if` `(v39` `=``=` `56``) {`

 `/``/` `*` `(& v5` `-``> REGS` `+` `(unsigned` `int``)v19)` `=` `*` `(& v5` `-``> REGS` `+` `(unsigned` `int``)v18) ^ (vm_code_1 &` `0xF000` `| (v21 >>` `20``) | (vm_code_1 >>` `6``) &` `0x3F` `| (v22 >>` `20``) | (v14 >>` `20``) | (v15 >>` `20``) | (v16 >>` `20``) | (v20 >>` `20``));`

 `/``/`     `}`

 `/``/`     `else` `if` `(v39` `=``=` `62``) {`

 `/``/` `*` `(& v5` `-``> REGS` `+` `(unsigned` `int``)v19)` `=` `*` `(& v5` `-``> REGS` `+` `(unsigned` `int``)v18) | vm_code_1 &` `0xF000` `| (v21 >>` `20``) | (vm_code_1 >>` `6``) &` `0x3F` `| (v22 >>` `20``) | (v14 >>` `20``) | (v15 >>` `20``) | (v16 >>` `20``) | (v20 >>` `20``);`

 `/``/`     `}`

 `/``/` `}`

 `/``/` `else` `{`

 `/``/`     `if` `(v39` `=``=` `4``) {`

 `/``/`         `LODWORD(v18)` `=` `(vm_code_1 &` `0xF000` `| (v21 >>` `20``) | (vm_code_1 >>` `6``) &` `0x3F` `| (v22 >>` `20``) | (v14 >>` `20``) | (v15 >>` `20``) | (v16 >>` `20``) | (v20 >>` `20``)) <<` `16``;`

 `/``/` `v18` `=` `(``int``)v18;`

 `/``/` `v25` `=` `&vm_regs[(unsigned` `int``)v19];`

 `/``/` `v25[``1``]` `=` `v18;`

 `/``/`     `}`

 `/``/`     `if` `(v39` `=``=` `32``)`

 `/``/` `*` `(& v5` `-``> REGS` `+` `(unsigned` `int``)v19)` `=` `*` `(& v5` `-``> REGS` `+` `(unsigned` `int``)v18) & (vm_code_1 &` `0xF000` `| (v21 >>` `20``) | (vm_code_1 >>` `6``) &` `0x3F` `| (v22 >>` `20``) | (v14 >>` `20``) | (v15 >>` `20``) | (v16 >>` `20``) | (v20 >>` `20``));`

 `/``/` `}`

 `var v39` `=` `code.``and``(``0x3F``)`

 `let opcode2` `=` `code.``and``(``0xF000``).``or``(Bit_30_State_1.shr(``20``)).``or``(code.shr(``6``).``and``(``0x3F``)).``or``(Bit_31_State_1.shr(``20``)).``or``(Bit_28_State.shr(``20``)).``or``(Bit_29_State.shr(``20``)).``or``(Bit_30_State.shr(``20``)).``or``(Bit_31_State.shr(``20``))`

 `if``(v39.toUInt32() >` `0x37``){`

 `if``(v39.toUInt32()` `=``=` `56``){`

 `return` `"eor "` `+` `this.reg(destREG)` `+` `","` `+` `this.reg(sourceREG)` `+` `","` `+` `opcode2`

 `}``else` `if``(v39.toUInt32()` `=``=` `62``){`

 `return` `"orr "` `+` `this.reg(destREG)` `+` `","` `+` `this.reg(sourceREG)` `+` `","` `+` `opcode2`

 `}`

 `}``else``{`

 `if``(v39.toUInt32()` `=``=` `4``){`

 `return` `"mov "` `+` `this.reg(destREG)` `+` `","` `+` `opcode2`

 `}``else` `if``(v39.toUInt32()` `=``=` `32``){`

 `return` `"and "` `+` `this.reg(destREG)` `+` `","` `+` `this.reg(sourceREG)` `+` `","` `+` `opcode2`

 `}`

 `}`

 `case` `0``:`

 `case` `2``:`

 `case` `8``:`

 `case` `0x12``:`

 `case` `0x1F``:`

 `case` `0x21``:`

 `case` `0x22``:`

 `case` `0x27``:`

[回复或点赞可查看完整内容](#quick_reply_form)

[[注意] 传递专业知识、拓宽行业人脉——看雪讲师团队等你加入！](https://bbs.kanxue.com/thread-275828.htm)

[#逆向分析](forum-161-1-118.htm)

上传的附件：

*   [demo.rar](javascript:void(0)) （99.20kb，17 次下载）