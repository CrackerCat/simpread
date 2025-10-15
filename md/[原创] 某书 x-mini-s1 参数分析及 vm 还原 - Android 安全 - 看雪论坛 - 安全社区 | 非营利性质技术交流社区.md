> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-288787.htm)

> [原创] 某书 x-mini-s1 参数分析及 vm 还原

一. VM 函数输入参数分析
--------------

vm 函数的输入参数为 32 字节的 hash 值  
![](https://bbs.kanxue.com/upload/tmp/786483_5C93TWVZUZB5J4S.webp)  
对该地址进行跟踪发现数据来自于 sub_127028 的计算  
![](https://bbs.kanxue.com/upload/tmp/786483_S2G7FUE2DZ7H7KT.webp)  
sub_127028 的输入为 url 参数 + deviceId + 固定的 sha256 值 + x-mini-mua 总计 0x4c9 字节  
![](https://bbs.kanxue.com/upload/tmp/786483_M2XUCVGEN3SMS77.webp)  
sub_127028 中的计算分为两步，第一步通过 sub_126D48 依次对 64 字节大小的消息进行依次计算得到最终的 sha256 值。  
![](https://bbs.kanxue.com/upload/tmp/786483_TEEPYJKDHRR2362.webp)  
sub_126E1C 函数则对不足 64 字节的消息进行填充并计算 sha256 值，该函数除了进行正常 sha256 填充之外，还会判断消息的长度，当大于 55 时会多进行一次 sha256 计算。  
![](https://bbs.kanxue.com/upload/tmp/786483_C8UWCACYN8DMG2Y.webp)  
然后根据输入的消息大小对末尾的部分字节进行填充，填充完成后进行最后一次 sha256 计算  
![](https://bbs.kanxue.com/upload/tmp/786483_VRW3BYE7ZQ96RQY.webp)  
完成 sha256 计算之后，每 4 字节转为大端序后作为 vm 函数的输入数据  
![](https://bbs.kanxue.com/upload/tmp/786483_NWVFAZHSE7ZY5CT.webp)

二. VM 基础框架结构
------------

### 1. 指令分发器

该 vm 的指令分发器仅有一个, 负责对所有指令进行分发执行，以下是具体代码及含义  
![](https://bbs.kanxue.com/upload/attach/202510/786483_XWNE44W7Y9TKR45.webp)

### 2. 虚拟寄存器

在 vm 当中少数几个物理寄存器用来执行特殊功能，以下是相关寄存器及对应功能  
![](https://bbs.kanxue.com/upload/attach/202510/786483_Z9NSMSTF9R3PCH4.webp)

### 3.vm 栈

该 vm 为栈虚拟机, 通过栈的 push/pop 操作进行数据计算，而与栈内存相邻的为局部变量区，局部变量不仅参与计算过程中的值的存储与读取，而且在调用子函数时也用于参数传递。  
![](https://bbs.kanxue.com/upload/attach/202510/786483_FXMUWDPQ8CZKHGK.webp)  
在 vm 入口处会根据当前函数需要的栈空间大小及局部变量数量进行内存的分配  
![](https://bbs.kanxue.com/upload/attach/202510/786483_UTVKGB27QNRRJT7.webp)

### 4. 流程控制

流程控制主要通过跳转指令 + 跳转表实现，跳转表中的元素结构如下  
![](https://bbs.kanxue.com/upload/attach/202510/786483_H6E2DCSA8KTDQKH.webp)  
流程控制指令分为两种类型，第一类为向跳转表压入指定元素，这类指令会根据当前指令所在虚拟 PC 判断是否执行跳转表写入操作。  
![](https://bbs.kanxue.com/upload/attach/202510/786483_27GDM5JC9F83AN7.webp)  
第二类为从跳转表中获取指定元素并跳转到指定地址，这些指令通过不同的操作实现了如 jne，jmp，switch 这些语句。  
jne 指令的实现如下  
![](https://bbs.kanxue.com/upload/attach/202510/786483_8VTJ59NY9RR7ZVW.webp)  
switch 语句的实现如下, 0x6F 为 op_code,08 为后面跟的数组元素最大值，然后为一个整数数组，每次 switch 语句获取栈顶数据作为整数数组下标，获取的值再作为下标从跳转表顶部向下查找到对于的跳转地址进行跳转。  
![](https://bbs.kanxue.com/upload/attach/202510/786483_AJZSK4XZHC55VMJ.webp)  
![](https://bbs.kanxue.com/upload/attach/202510/786483_JTE5V39P3TS3K64.webp)

### 5. 内存的读写

指定内存的读取与写入通过基地址 + 偏移的形式，不同的指令会对应不同的基地址  
以 86 02 E0 02 为例，各部分数据的含义如下  
![](https://bbs.kanxue.com/upload/attach/202510/786483_AA7F4M5ZCC2JNZR.webp)  
然后指令对应的执行代码如下  
![](https://bbs.kanxue.com/upload/attach/202510/786483_N9CF3UDBVM3TGRN.webp)  
以上代码可以概括为以下伪代码，0x41100000 就是该赋值指令隐含的基地址

```
pop a1;
pop a2;
[0x41100000+a2 +0x160] = a1;  

```

三. VM 指令功能解析
------------

为了对 vm 指令进行等价还原，我们需要解析出每条指令的具体操作，重点在于对栈和内存的读取写入操作。根据 unidbg 的 trace 文件结合指令分发器代码，统计出参与 x-mini-s1 参数运算的指令有 44 种, 我们需要逐条分析这 44 条指令的功能。

### 1. 运算指令

运算指令包括加减乘除, 移位，比较等指令, 这类指令的 handler 一般代码短，逻辑清楚易分析，但需区分有符号运算与无符号运算。  
以下为加法运算的 handler  
![](https://bbs.kanxue.com/upload/attach/202510/786483_C5SUUHNDSCQYF4M.webp)  
以下为无符号比较 ne 的 handler  
![](https://bbs.kanxue.com/upload/attach/202510/786483_424XTRMMF3GGC6M.webp)

### 2. 赋值指令

赋值语句逻辑基本相同，但需要注意读写的数据长度  
以下为指令 46 00 87 05 的 handler, 该指令写入的内存地址偏移 = 指令中的立即数 leb128(87 05) 即 0x287 + 栈数据 a2, 并且写入的数据长度为 1 字节。  
![](https://bbs.kanxue.com/upload/attach/202510/786483_E7Y367C2GQRCM35.webp)

### 3. 流程控制指令

这类指令通常涉及对跳转表的读取及写入操作, 因此在分析过程中发现跳转表相关操作时，基本可以认定为流程控制相关指令。  
以下为 80 00 指令的 handler  
![](https://bbs.kanxue.com/upload/attach/202510/786483_XGZBJ77FYQCD5C6.webp)

### 4. 外部函数调用指令

对于某些比较复杂的操作或者可以直接调用系统函数获取的功能，外部函数调用指令负责传递参数并获取结果。  
这类指令从虚拟机进入外部函数的入口统一，并且入口函数被 ollvm 混淆，所以往往在 trace 文件中具有大量的代码。  
以 60 84 80 80 80 00 指令为例，该指令与下一条指令间相隔了约 4000 行汇编代码，我们要逐条分析无疑是十分困难的  
![](https://bbs.kanxue.com/upload/attach/202510/786483_K2KSQ8XEMPSRWNX.webp)  
因此对这类指令的分析我们可以利用 unidbg 的内存监控功能，查看指令执行前后的内存读写结果，对外部函数功能进行推测。  
对 60 84 80 80 80 00 指令监控后发现每次返回的数据都不相同，推测该外部函数的作用为返回随机数  
![](https://bbs.kanxue.com/upload/attach/202510/786483_VMYMCAXTRB9KTN5.webp)  
![](https://bbs.kanxue.com/upload/attach/202510/786483_JAVB6UUXJGFSZQA.webp)  
而随机数的生成一般使用当前时间作为随机种子，为了验证我们的猜测, 我们可以固定 unidbg 中的当前时间  
![](https://bbs.kanxue.com/upload/attach/202510/786483_AH4GKN7AB859RVM.webp)  
再次执行发现返回值已固定，因此确认 60 84 80 80 80 00 指令功能为获取随机数  
![](https://bbs.kanxue.com/upload/attach/202510/786483_PZSTSABXZU55MDS.webp)  
![](https://bbs.kanxue.com/upload/attach/202510/786483_JS5NNSKZGK2UAVP.webp)

四. VM 还原
--------

为了对 vm 执行逻辑进行还原，我们需要对 vm 的以下各个部分进行模拟  
(1)vm 中的基础组件如 vm 栈，内存读写  
(2) 控制流的模拟，vm 中的控制流基于跳转表及各类控制指令  
(3)vm 指令  
为了对以上各部分进行模拟，也为了能够将最终还原的指令进行编译优化，生成二进制文件从而利用 ida 进行分析，我选择的是 llvm ir 进行模拟，并且 llvmlite 作为 llvm ir 的 python 接口，使用起来更方便高效。

### 1.vm 栈的模拟

模拟 vm 栈可以使用 lvm ir 先分配一块空间作为栈空间  
![](https://bbs.kanxue.com/upload/attach/202510/786483_CWVEEH9CGQ3EVTS.webp)  
然后定义 push/pop 操作, 并且维护好栈顶指针  
![](https://bbs.kanxue.com/upload/attach/202510/786483_AKUJ8ZK7NEGZ6K2.webp)

### 2. 控制流模拟

对控制流模拟可以还原出各个代码块之间的逻辑关系，但如果对跳转表组件进行模拟的话不仅需要在内存中维护一个跳转表，还需要解析 vm 中控制流相关数据结构, 以正确执行插入跳转表的操作，这无疑是不小的工作量，并且这样模拟之后代码块之间的跳转也是动态的，无法直观的显示各个块的逻辑关系。  
我们的目标是还原执行逻辑，而执行过程中每一条 vm 代码的执行都可以被记录在 trace 文件当中，所以我们可以利用 trace 文件获得完整的执行流，而根据前面对于指令的解析我们可以识别出控制流相关指令，根据这些指令我们就可以将执行流区分为不同的代码块，并且这些代码块之间的跳转关系也一目了然。  
![](https://bbs.kanxue.com/upload/attach/202510/786483_89FRJGEK9RAA4XD.webp)  
根据以上方法，我们获得部分控制流图如下, 可以清晰的看见其中的 switch，循环，条件跳转等逻辑。  
![](https://bbs.kanxue.com/upload/attach/202510/786483_X7PJRWCZZZ7SHEP.webp)

### 3. 内存读写模拟

对于内存读写模拟，我们也使用与 vm 栈模拟类似的方法，先分配一块内存  
![](https://bbs.kanxue.com/upload/attach/202510/786483_CV5ER6WEHPEU2AJ.webp)  
由于 vm 中对于内存的模拟是默认基地址 + 偏移的形式，所以我们也需要定义相应的读写函数，输入参数包含偏移地址和数据大小。  
![](https://bbs.kanxue.com/upload/attach/202510/786483_QJB56NQJUBUV8CC.webp)  
![](https://bbs.kanxue.com/upload/attach/202510/786483_UKDD8Y5TCB88KYW.webp)

### 4. 指令模拟

#### (1) 运算指令

类指令的模拟只需要根据 pop 操作获取到数据，然后执行完运算之后将结果重新 push 到栈中  
以下是 vm 中的 lsl 指令对应的 llvm ir 生成代码  
![](https://bbs.kanxue.com/upload/attach/202510/786483_4FPENYG86RZH4E8.webp)  
以下是有符号比较 < 对应的 llvm ir 生成代码  
![](https://bbs.kanxue.com/upload/attach/202510/786483_957SRVKRYBQ7EUD.webp)

#### (2) 赋值指令

赋值指令基于我们之前实现的内存读写模拟也可以轻松实现  
以下是前文描述的 86 02 E0 02 等内存读写指令对应的 llvm ir 生成代码  
![](https://bbs.kanxue.com/upload/attach/202510/786483_6HK4HA7XA8VQAGT.webp)

#### (3) 流程控制指令

之前我们已经根据 trace 文件生成了控制流图，找到了各个代码块之间的上下文关系，并且根据流程控制语句的类型我们需要进行不同的处理。  
在 llvm ir 中所有块之间都要有明确的跳转指令，如果两个块之间的关系是直接 jmp 类型，那么我们在一个块的末尾就需要加上强制跳转指令。  
![](https://bbs.kanxue.com/upload/attach/202510/786483_HY4F9ZG4PCWQCT9.webp)  
如果是 jne 指令，我们需要在 ir 中加上从栈中 pop 数据并且判断值再跳转这个过程  
![](https://bbs.kanxue.com/upload/attach/202510/786483_WVJYSCSAFVSSWM3.webp)  
如果是 switch 语句的话相对麻烦一些，因为我们需要找到执行不同的 case 时对应的变量值才能建立映射关系，我们可以在 trace 文件中查找变量的读取语句然后与后续执行的代码块地址进行匹配，也可以在 unidbg 中关键节点下断点读取出指定的变量值与代码块匹配。这两种方法都存在由于输入数据的局限性导致部分 case 未执行的情况，因此我们可以尽量提高数据的多样性使其充分执行，当前也可以遍历跳转表中的数据拿到所有 case 下的跳转代码块地址。  
![](https://bbs.kanxue.com/upload/attach/202510/786483_32FJZEPN8P2JSY5.webp)

#### (4) 外部函数调用指令

对于外部函数的调用，如果 llvm ir 中已经有的现成函数如 memset，memcpy 等我们可以直接调用即可，而其他函数如 time 函数或者自定义函数我们可以仅创建对应名称的函数进行调用即可, 而如随机数函数直接将其固定的值压入栈中即可，不创建函数声明。  
![](https://bbs.kanxue.com/upload/attach/202510/786483_B8RCZM7PVQN6J2X.webp)

### 5. 生成. o 文件

完成以上流程之后我们就可以生成所有 vm 指令对应 ir 代码的文件  
![](https://bbs.kanxue.com/upload/attach/202510/786483_GVPV6A6764CKKRS.webp)  
然后通过以下指令即可编译为 aarch64 架构的二进制文件，其中 x-mini-s1.ll 为生成的 ir 代码文件，而 section.ld 文件用于将我们模拟的内存固定到指定的基地址，方便后续分析代码逻辑时与 unidbg 进行交互验证。

```
clang -target aarch64-unknown-linux-gnu -shared ./x-mini-s1.ll  -o x-mini-s1.o -T ./section.ld

```

```
SECTIONS
{
    .datasection 0x41100000 : {
        *(.datasection)
        }
 
    .text 0x400000 : {
        *(.text)
        }
 
    .data : {
        *(.data)
        }
 
    .bss : {
        *(.bss)
        }
}

```

最终生成的二进制文件反编译之后如下所示，可以看见虽然还是有大量内存操作，但是逻辑十分清晰, 达到了可以人工分析的程度。  
![](https://bbs.kanxue.com/upload/attach/202510/786483_PHG2JK27YZXD259.webp)

### 6. 分析. o 文件

生成二进制文件之后，就可以分析 x-mini-s1 的生成逻辑了, 由于 vm 函数的输入随着每次运行都会变化，为了方便在 unidbg 中调试，我们可以先在 vm 函数入口处固定输入。  
![](https://bbs.kanxue.com/upload/attach/202510/786483_GKPNYAENH8M2VMV.webp)  
在分析过程中编写等价代码时往往需要快速对比 unidbg 中的内存数据来判断是否正确，但在 vm 中要定位到具体的针对特定地址的读写操作十分的麻烦，因此可以在内存读写模拟代码中添加打印功能，显示出不同地址指令的读写操作。  
![](https://bbs.kanxue.com/upload/attach/202510/786483_5XW5S842X6QWD8M.webp)  
比如我们想查看以下循环执行之后的内存, 就可以在 dword_4116FE9E 被赋值时下断点  
![](https://bbs.kanxue.com/upload/attach/202510/786483_AEN2MJ454FD39QV.webp)  
通过打印信息我们查找到在 vm pc 为 0x172e 时对该地址进行了赋值，所以我们可以在指令分发器中过滤该 pc 地址  
![](https://bbs.kanxue.com/upload/attach/202510/786483_JXEA8K3B2X3R6HM.webp)  
最终将所有代码使用 python 还原之后可以生成与 unidbg 相同的 x-mini-s1 值  
![](https://bbs.kanxue.com/upload/attach/202510/786483_PFE2DN6WG4U5XVK.webp)  
![](https://bbs.kanxue.com/upload/attach/202510/786483_8FCW3Q6YTXW8A5C.webp)  
![](https://bbs.kanxue.com/upload/attach/202510/786483_76YAB957F76WBHC.webp)

[[培训] 传播安全知识、拓宽行业人脉——看雪讲师团队等你加入！](https://bbs.kanxue.com/thread-275828.htm)

[#协议分析](forum-161-1-120.htm) [#逆向分析](forum-161-1-118.htm)