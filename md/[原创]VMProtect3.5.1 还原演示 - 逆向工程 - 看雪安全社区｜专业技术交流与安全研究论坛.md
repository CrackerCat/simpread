> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-293303.htm#msg_header_h1_7)

> 看雪安全社区是一个非营利性质的技术交流平台，致力于汇聚全球的安全研究者和开发者，专注于软件与系统安全、逆向工程、漏洞研究等领域的深度技术讨论与合作。

**VMProtect3.5.1 还原演示**

Author: malware/zhouqing Date:2026/8/20 Title:Agent 能力的演示

  在早期 VM 攻防当中，红方通常采用手写反汇编器解析 OpCode 做解释器的手段对被 VM 保护的程序进行分析。在 2019 年，一篇名为《SATURN》的论文发表于第 3 届 ACM 软件保护研讨会，证明了一种不依赖特定 VM 语义知识的通用反混淆路径的可行性。

 《SATURN》基于 Remill 将 Binary 抬升至 LLVM IR 后采用 DFS 路径遍历算法恢复 CFG（二进制控制流图）后利用 LLVM 内置优化 Pass 和 Souper 超级优化器，消除常量展开、死代码插入、基于算术的不透明表达式等常见混淆模式。对于优化无法解决的不透明谓词，则交由 SMT 求解器（Z3）处理。然后将 Remill 特有的 State 结构体操作转化为普通函数参数和局部变量，恢复栈和函数参数，使代码接近源码编译结果。（基于 ABI 的调用约定逆向与寄存器 / 栈槽的语义重构）

![](https://bbs.kanxue.com/upload/attach/202610/960900_NBBGKVBKSUB5EMW.webp)  

  其中，Remill 是由安全公司 Trail of Bits 开发和维护的开源项目，代码托管在 GitHub 的 lifting-bits 组织下。根据《SATURN》的测试，Remill 与 McSema、RetDec、MCTOLL 等现有二进制提升框架对比，这些工具在面对混淆代码时均失败。与 Triton（符号执行工具）对比，《SATURN》生成更小、更优化的 SMT 查询，求解效率更高。

  在 7 月 21 日的时候，我在 linkedin（领英）上发现了一篇关于 VMProtect 逆向的推文，“Static devirtualizer for VMProtect 3.0-3.5. Lifts virtualized code to LLVM using Remill and strips the VM layer through optimization.”

  经过对比，发现其思想与《Saturn》一致，旨在不对 VM 保护程序进行针对性的指令集架构处理，尝试还原出程序当中原始的运行逻辑。作者开发的 MogVMP 利用 remill 的抬升能力，抬升一个 handler 编译解密出下一个 handler 地址循环抬升解密，并利用 LLVM 的 alloca 特性使得 VMProtect 的寄存器轮转等内容在编译时自然地被消除。在还原 VMProtect 基于 vJmp 与 flag&mask 实现的条件跳转时，利用 LLVM 提供的变量污点分析能力还原分支跳转。在针对 VMProtect 当中基于 rdtsc 时间指令反调试时同样利用 LLVM 的污点分析能力对 remill 产生的变量进行活跃分析最终实现反反调试。

![](https://bbs.kanxue.com/upload/attach/202610/960900_7JSVWVW4BR54TFC.webp) 

![](https://bbs.kanxue.com/upload/attach/202610/960900_W7KXY42ZWW274MD.webp) 

  同时，看雪论坛 2025 年 12 月的一篇技术文章《深入浅出 Ollvm 混淆原理及反混淆技术》，揭示了一种基于动态执行的 OLLVM 反混淆还原方法。该方法基于 Unicorn 的动态模拟执行机制，实时捕获程序对关键数据结构的读写操作，辅助分析混淆后变量的真实语义。

![](https://bbs.kanxue.com/upload/attach/202610/960900_DWYDUKAZRBR5HRP.webp) 

  2026 年 7 月 20 日，一款名为 AVLCode 的 AI 智能编程与安全分析桌面助手出现在密界的视野当中，其搭载的澜砥威胁检测分析垂直大模型由安天自主研发，面向二级制、脚本等对象进行分析鉴定，突破开源模型局限，将检测文件的大小扩展到数百兆字节级别 (亿 Token 上下文)，在处理二进制数据时提供 500~1000 倍的性能。

  澜砥大模型的主干模型 VILLM（Virus Inspection Large Language Model）可处理二进制字节数据、进行威胁检测分析，支持最长 336M 的超长上下文分析。另外两个分支模型 SecOpsGPT（大安）侧重自然语言处理与多模态数据理解、SecDevGPT（小元）侧重程序代码补全，分支模型基于开源模型改造并使用社区权重进行迁移学习训练，分别用于安全运营辅助和安全开发辅助。

![](https://bbs.kanxue.com/upload/attach/202610/960900_D9R74UMJSXK9JPB.webp) 

  2023 年 9 月 8 日，xx_vm 的作者 xxidsasm 开放了插件源码的整套开放合购，该插件是目前市面上公开针对 3.x 能力最好的插件，早期 Ollydbg 版本是基于 Dynamic 物理机动态 Trace 的，在 X64Dbg 版本时取消了 Dynamic，实现了静态 Static 的模拟执行。

![](https://bbs.kanxue.com/upload/attach/202610/960900_AWTKZ59WD2F5AM5.webp) 

![](https://bbs.kanxue.com/upload/attach/202610/960900_AZG4U7C9NAG36HP.webp) 

  因为预期目标是还原 Trace 部分的逻辑，所以相对合适的构造方式是利用古典密码学的置换代换的思想构造串联的双射空间，这能够让 Trace 覆盖相对多的算法部分。输入一段提示词，让 deepseek 生成代码，具体代码见附录：

用 C 编写一个 Crack Me，调用 MessageBoxW 告诉用户是否验证成功，反复思考一下 CheckSerial 的设计，我的期望是一个古典密码学，通过一个 key 进行轮转。

- 从简单的置换（如 Scytale 密码）到单表代换（如 Caesar 密码），再到多表代换（如 Vigenère 密码），复杂度逐步提升。

预期的结果是 flag{bbs.kanxue.com}

  在三个函数的内部添加 VMPSDK，而具体的 CheckNameSerial 则并不添加 VM 保护。这创造了一个选择明文攻击条件，便于我们快速观察预期的 En(flag)，就像一道 CTF 题目一样。蓝方期望红方还原程序当中被混淆的算法而暴露出预期的密文以及针对用户输入而产出的 En(serial) 是怎样的，方便红方判断算法还原的准确性。

![](https://bbs.kanxue.com/upload/attach/202610/960900_K22ENWM7A28QCD7.webp) 

  同时在此给出这三个函数所分别对应的加密等级：

<table><tbody><tr><td><p>函数名</p></td><td><p>等级</p></td><td><p>备注</p></td></tr><tr><td><p>scytale_permutation</p></td><td><p>Ultra（超级虚拟化）</p></td><td><p>置换</p></td></tr><tr><td><p>caesar_shift</p></td><td><p>Virtual（虚拟化）</p></td><td><p>单表映射</p></td></tr><tr><td><p>vigenere_encrypt</p></td><td><p>Virtual（虚拟化）</p></td><td><p>多表映射</p></td></tr></tbody></table>

  将 remill 的源码 clone 到本地后，准备一个 LLVM（version 17+）的编译环境将其编译好作为静态削弱混淆的基础插件。然后我们编写一个 x64dbg 的动态调试插件能够支持 Dump、Trace 功能，我将其取名为 malware。当前，我们的分析环境如下：

<table><tbody><tr><td><p><strong>工具名</strong></p></td><td><p><strong>备注</strong></p></td></tr><tr><td><p>LLVM 17 源码及工具链（Windows 平台）</p></td><td><p>编写 pass/opt 优化 / 重编译程序</p></td></tr><tr><td><p>基于 Unicorn 和 Zydis 的<br>X64Dbg 动态调试插件</p></td><td><p>保存程序动态执行路径及数据</p></td></tr><tr><td><p>二进制抬升器 Remill-lift-17.exe（Windows 平台）</p></td><td><p>用于将 Binary 抬升至 LLVMIR 生成. ll</p></td></tr><tr><td><p>AVLCode/VLLM（澜砥大模型）</p></td><td><p>辅助分析混淆规则</p></td></tr></tbody></table>

接下来，我通过 Trace 插件进行程序控制流的跟踪，值得一提的是 until address 会根据 ebp 智能计算目标地址，但是由于程序似乎并非 MSVC 编译，不依赖 mov ebp,esp 的方式进行栈帧的管理，导致这个方法失效了，需要手动填写目标地址。

同时可以发现函数头并没有被 VM，这是 VMP SDK 所带来的弊端。通过没有被 VM 掉的函数头，我们向 AVLCode 询问其函数的参数。

![](https://bbs.kanxue.com/upload/attach/202610/960900_EYGE4PCFNPAV6B8.webp) 

  我们向 AVLCode 询问，创建好工作区后，输入提示词：

目标是 x86 32bit 程序，程序似乎并非 MSVC 编译，请帮我判断一下该函数的局部变量数量和参数：

-----------

函数头部汇编代码

-----------

  最终得到局部变量的 stack point 大小 (0xa0) 和俩个函数参数的信息：函数入口指令 sub esp, A0h 表明该函数分配了 0xA0（160）字节 的栈空间。

在 x86 32 位调用约定（cdecl/stdcall）下，栈布局（从低地址到高地址）为：

<table><tbody><tr><td><p>偏移 (相对于 esp)</p></td><td><p>内容</p></td><td><p>说明</p></td></tr><tr><td><p>[esp+0x00] ~ [esp+0x8F]</p></td><td><p>局部变量区域</p></td><td><p>约 144 字节的临时存储空间</p></td></tr><tr><td><p>[esp+0x90]</p></td><td><p>保存的 ebx</p></td><td><p>注释显示为字符串指针 "aaaaaaaaaaaaaaaaa"</p></td></tr><tr><td><p>[esp+0x98]</p></td><td><p>保存的 edi</p></td><td><p>注释显示为路径字符串指针</p></td></tr><tr><td><p>[esp+0xA0]</p></td><td><p>返回地址</p></td><td><p>call 指令压入</p></td></tr><tr><td><p>[esp+0xA4]</p></td><td><p>参数 1</p></td><td><p>注释显示为 "aaaaaaaaaaaaaaaaa"</p></td></tr><tr><td><p>[esp+0xA8]</p></td><td><p>参数 2</p></td><td><p>被加载到 ebx</p></td></tr></tbody></table>

![](https://bbs.kanxue.com/upload/attach/202610/960900_EMY72HNKEV3Y6YM.webp) 

  在掌握函数头部所带来的信息后，我们来检查一下 Trace 后的产物，004015E9 产生了 3 次 Trace，这说明了程序共有 3 次执行到了. vmp0 section（区 / 节）之外的地址然后被我们的动态调试插件所捕获。同时注意到第二份和第三份的 Trace 文件相比第一份，所占用的磁盘空间 bytes 每一次都在上升一个量级。

![](https://bbs.kanxue.com/upload/attach/202610/960900_5DAHZVS7AZK4GQU.webp) 

  相应地，这也意味着反混淆的成本也在呈量级的增加，整理一下 004015E9 的 Trace 文件与现有的信息，我们可以构造出这样的控制流：

0XA0 个活跃变量，2 个函数参数，一个指向了 serial（用户输入 ebx）一个则是一个 const 常量 0x3（edx）

0x004015E9 ~ 0x00408F50（一段未知的算法）

Strlen

0x00433678 ~ 0x00408F38（一段未知的算法）

Strcpy

0x0047B955 ~ 0x0046E27D

0x004090D7（.text 段代码，说明 VM 执行结束了）

  关于分析环境的介绍就到此结束了，我们当前掌握了一个函数的参数信息，和俩个实际调用的 api。接下去则是去分析 Trace 当中的 VM 了。

  针对 Trace 当中所存在的死代码。通常情况下要做个活跃变量分析，通过将控制流产生的 SSA 数据流来判断什么变量在被读取前就被二次写入了，进而消除掉未被利用的代码。我们将这样的一种过程称之为死代码消除。

而常量折叠在控制流的体现上则是将未知的跳转，例如 jmp reg，通过传递 state 和 memory 等数据经过 LLVM 后自动削弱掉不必要的计算，最终产生一个常量目标地址。同时一种相对常见的处理方案是利用 Triton 等符号执行框架去计算可行的路径，最终算出一个常量目标地址。

![](https://bbs.kanxue.com/upload/attach/202610/960900_YCX9T4PUJDVQVT4.webp) 

  在 LLVM 当中，-O1 优化参数对应着常量折叠、死代码消除。它是如下指令的缩写，接下来我介绍一下各个参数的含义。

opt -S -dce -simplifycfg -constprop -gvn raw.ll -o optimized.ll

各个参数的含义如下：

<table><tbody><tr><td><p><strong>Pass</strong></p></td><td><p><strong>作用</strong></p></td></tr><tr><td><p>-dce</p></td><td><p>死代码消除：删除从未被使用的指令。</p></td></tr><tr><td><p>-simplifycfg</p></td><td><p>控制流简化：合并基本块、消除空块、简化分支。</p></td></tr><tr><td><p>-constprop</p></td><td><p>常量传播：将可静态确定的计算（如 add i32 1, 2）折叠为常数。</p></td></tr><tr><td><p>-gvn</p></td><td><p>全局值编号：识别并消除重复的计算。</p></td></tr></tbody></table>

  针对我们的第一份 Trace，我们利用 remill 将其抬升至. ll 文件，使用上述的 - O1 优化，这个. ll 文件比较大，有 500 多 MB。然后我们利用 llc(LLVM 工具链中的静态编译器后端) 将. ll 链接到. obj 文件，然后用 IDA 打开，F5 反编译查看，可以看到，我们的 vm_entry 很干净，vm_entry 入口的 key 丢失了，给 vm_context 区域设置为 0 后执行 strlen 退出 VM，因为. ll 没有拼接起来所以链接出的. obj 反编译看上去 vm_ret 像是在死循环。

总之，直接利用 LLVM 削弱 VM 函数，会导致几乎全部的信息丢失，这就要求我们构建专门针对 VMProtect3.5.1 的 LLVM 工具。或者把虚拟机的执行过程当作普通程序代码，整体通过翻译器例如 remill 翻译成 LLVM IR，再让编译器优化器把 “虚拟机层” 当作冗余代码折叠掉。

![](https://bbs.kanxue.com/upload/attach/202610/960900_5SUQT84WHW2UR4T.webp) 

  针对 VMProtect Ultra 当中的混淆函数，针对 32bit 的寄存器操作拆分成了俩个 16bit 寄存器或多个 8+16bit 的 MBA 混淆运算，基于此利用 LLVM ElementPTR 的特性建模，形如：

%physical_eax = getelementptr inbounds %struct.State, ptr %state,

          i32 0, i32 0, i32 6, i32 1, i32 0, i32 0

%physical_ebx = getelementptr inbounds %struct.State, ptr %state,

          i32 0, i32 0, i32 6, i32 3, i32 0, i32 0

而 physical_ah（高位）physical_ax physical_al（俩个低位 8bit） 则是 physical_eax 的高位和低位寄存器，一个 32-bit guest register 是 State 中的一个 32-bit storage slot，而 LLVM 允许通过 GetElementPTR + typed load/store 从这个 slot 的不同 byte/word offset 建模 AL/AH/AX 等子寄存器视图，从而利用 TBAA / Alias Analysis / MemorySSA 等分析削弱混淆。

  在我们的第四章第一节当中介绍了常量折叠和死代码消除，我们描述了一种通过 LLVM 直接 O3 优化混淆函数导致信息全丢的现象，这里有一个办法是 LiveOut 建模去欺骗编译器。

  例如，我们要求能够在混淆代码中识别 vip、vmcontext、vmkey 等符号后，编写代码标记其在 LLVM 函数（handler）结束后，生命周期依然是有效的。而其他函数过程产生的中间变量设置 undef 属性，要求 LLVM 在最终要对其进行常量折叠和死代码消除。

最终，可以看到下图的清理效果，同时我们可注意到 xor ecx,ebx 并没有被优化掉，因为 ebx 是 vmkey，用于解密下一个 handler 的地址。

![](https://bbs.kanxue.com/upload/attach/202610/960900_E7QHEVV4A5WPT33.webp) 

  通过 LiveOut 活跃变量建模欺骗编译器本质上依赖我们对现有数据建模的准确性，对此，VMProtect 团队也不曾多让，设计了 “寄存器轮转” 这一机制动态的切换寄存器所代表的 vm runtime state（VM 运行时状态）。

  为了解决寄存器轮转这一问题，当前常见的解决方案是利用符号执行技术针对 handler 间内部利用的 reg 进行污点分析，当 reg 发生变化时尝试溯源构建寄存器轮转后变化后的映射关系。

  remill 把 x86 的全部架构状态建模为一个 C 结构体 X86State，对于 32 位程序的二进制提升 IR，GPR 数组不只存 EAX-EDI（8 个物理寄存器），同时 EIP 作为 GPR[33]。把寄存器当作 "指针" 统一管理，这与我们此前关于 VMContext 建模当中介绍的 ElementPTR 一致。

  而在本案例当中，我们自身也提供了 Trace，基于此，LLVM 的 O3 自身就可以优化关于 EIP 寄存器的操作，只保留基本块尾部的下一个基本块。借此强调指针这一概念的重要性，GEP 把寄存器的状态变成可寻址的内存位置，LLVM 的别名分析和常量传播自然接管了存活判断，不需要我们手写规则去删 physical_eip 的 store。

![](https://bbs.kanxue.com/upload/attach/202610/960900_MVX98YWPFVJ8WFZ.webp) 

  在 remill 的 State.h 源码中，存在着这样的 GPR（General-Purpose Registers / 通用寄存器）结构，其与我们的物理寄存器映射关系如下：

![](https://bbs.kanxue.com/upload/attach/202610/960900_FV2DB8R7CXJVT7X.webp) 

  其中，Reg 结构体用于以多种位宽访问同一个物理寄存器，精确模拟 x86 CPU 的寄存器访问行为。第二个版本可能是为了兼容 32 位二进制模式而做的调整。（_padding0 用于确保在 32 位和 64 位模式下结构体大小相同，同时二进制抬升器以 32 位模式运行时，防止意外访问高 32 位数据。

![](https://bbs.kanxue.com/upload/attach/202610/960900_DD654ACGA8FCJV3.webp) 

  基于此，我们知晓了该版本 remill 结构体，可以针对示例代码做如下解释，还原 .ll 文件当中 IR 关于 GPR（通用寄存器）的映射关系：

![](https://bbs.kanxue.com/upload/attach/202610/960900_NDVUEU92WE9JSBB.webp) 

  至此，我们已经成功削弱 Mutation 混淆，至少在针对 VM 建模正确时它是有效的。

  在 zvrop 的 2012 zvm 与天扬的 2013 VMP 分析插件 1.4 基础上完善了 VMP 特征识别功能。

  在此将 EAX、EBX 等 32bit 寄存器统一为 Reg4, 与 天杨的 2013 VMP 分析插件 1.4 命名风格一致，ax、bx 等 16bit 寄存器统一为 Reg2, ah、al 等 8 bit 统一为 Reg1。一般地，经过 LLVM 激进死代码消除与常量折叠（无 GVN）后，.ll 会将 handler 行为暴露出来，还原回汇编指令可借此匹配。非预期情况做符号执行。

同时，我们需要讨论一下 opcode 方向的问题，在被 VM 的函数当中，vEIP 在 opcode 的执行顺序有时是线性递减而非递增的，这需要在识别出 VMEntry 时判断出方向添加到 VMContext 当中。

  一旦 opcode 方向是通过 vEIP 递减遍历的，就要求匹配 handler 的时候，关于所有 handler 的 ADD vEIP, n 都要替换为 SUB vEIP, n。SUB vEIP, n 替换为 ADD vEIP, n。

![](https://bbs.kanxue.com/upload/attach/202610/960900_Z62W2U4DS8DFD6P.webp) 

  在 VMP3.5.1 条件下，vKey 参与 handler 地址解密、handler 参数解密，handler 尾部跳转下一条 handler 前，用于解密 vKey 的 vEIP 是不会发生变动的。在我们已经做好了 handler 的匹配的基础上，通过追踪 vEIP 的数据流，基于 Triton 做符号执行拿到参数。如下方的 vPopReg4 0x24。

![](https://bbs.kanxue.com/upload/attach/202610/960900_3DKMYZQUMFM4PBD.webp) 

  根据《VMProtect 的逆向分析和静态还原 -2009 中国软件安全峰会》VMP 将所有寄存器都存放在了堆栈的结构中（VM_CONTEXT），结构中的每一项代表一个寄存器或者临时变量。但在运行过程中，其中的项所映射的真实寄存器都是不固定的，可以把它比作一个齿轮，每做完一个动作，部分项的映射就互换了一下位置，或者执行完一段指令，齿轮就按不固定的方向和度数转动一下，然后全部的项映射就改变了。VMP 在生成字节码的过程中，维护了一份结构中每一项所映射的真实寄存器，但这只存在于编译过程，而在运行时是没有明确的信息的。这直接导致了分析和识别的难度。

在此基础上，寄存器轮转的概念可细化为物理寄存器的轮转和 VMContext 的槽轮转，物理寄存器的轮转在 VM 的表现为 vESP、vEIP 所对应的物理寄存器发生变化，例如在早期 VMP 当中，vESP 通常对应着 ebp 物理寄存器而 vEIP 通常对应着 esi 物理寄存器。

  在本案例，这样的轮转过程发生在 vJmp 当中，具体的 vEIP 寄存器轮转过程参见下图 vJmp 的行为分析，由 ebp 变化为了 edi，这就导致前文所介绍的 handler 模式匹配与参数还原失效。因为模式匹配依赖 vEIP、vESP 所识别出的物理寄存器是正确的。

由于通用物理寄存器的数量有限，排除在 x86 处理器下默认的 esp，vEIP 和 vESP 的轮转选择只有 7 个。

![](https://bbs.kanxue.com/upload/attach/202610/960900_4YBDQKF7KBTY4D8.webp) 

  相应地，存在 VMContext 的槽轮转，这是一种针对 VM 虚拟化出来的上下文结构的一种混淆方式，3.x 寄存器轮转不依赖 vNor，在编译时已经规定好轮转的位置 vEIP 通过 vJmp 轮转，11 个 slot 通过 vPushReg4 抬到栈顶后通过 vPopReg4 重新分配 slot 位置，如果考虑去除 VMContext 结构，还原为对原始的物理寄存器操作必然需要对 11 个槽进行污点，还原原始物理寄存器映射。

![](https://bbs.kanxue.com/upload/attach/202610/960900_22Q3CH28P5TFVUJ.webp) 

  在分析了 Handler 的模式匹配，还原参数，追踪完物理寄存器轮转与 VMContext 的槽轮转后，可还原为 VMP 的虚拟化指令集语义，同时提供 VMEntry 时的寄存器映射。

![](https://bbs.kanxue.com/upload/attach/202610/960900_N7XR4CT8H93XUAK.webp) 

  在我们还原了指令序列、物理寄存器轮转、寄存器初始化时的 VMContext 槽位映射关系的基础上，我们可以写个解释器去模拟 VM 栈机，去跟踪 VMContext 的槽轮转以及还原 VMContext 槽未所对应的物理寄存器操作，这里简单介绍一下我们的栈机模拟工具，模型主要由槽位偏移、槽位索引（偏移 / 4）、身份（对应的物理寄存器）、槽位 ID（按照槽位偏移升序（0..N-1）四部分构成。该工具最终会还原每个 VM 指令的参数符号化来源以及对应的物理寄存器（无法识别的情况为?，该情况为占位空闲 / 不稳定槽，后续会被二次覆盖），具体栈机模型如下图所示：

![](https://bbs.kanxue.com/upload/attach/202610/960900_2UV7BX98AVFVXXE.webp) 

  我们的样本有个 scytale_permutation 函数，这是我们在还原时会遇到的第一个 VM 函数，现在来尝试还原第一个 VMTrace，该函数会产生 3 个 VMTrace，意味着 VMP 三次进入虚拟机，三次退出完成目标。总计 3 个 VM 函数。

  我们可以看到在 VMEntry 后初始化完 VMContext，有一系列关于 vESP 的操作代码，由于我们的栈机模拟工具追踪了 handler 的完整参数符号化来源，所以很多代码都不需要去看，因为那是 MBA 混淆的一部分。只需要看最后一个有效的 handler 就可以观察完整的 MBA 混淆后的代码了。同时注意到 VMKeyRet（VM 在 push vKey call xxx）时产生的返回地址没有被用到，这意味着 VMKeyRet 这个槽是无用槽，可直接被优化掉。

![](https://bbs.kanxue.com/upload/attach/202610/960900_TQCXJNZ6Z9M8XZU.webp) 

  根据 Trace 日志，vESP 经过 MBA 混淆没有任何变化，这一大段都是死代码，可通过常量折叠和死代码优化消除，这些槽位后续都被二次覆盖了。

![](https://bbs.kanxue.com/upload/attach/202610/960900_7SM8GCXYNUTFM3M.webp) 

  随后，我们看到了第一个关于物理寄存器的读写操作，我们可将其还原为 sub esp,4；mov [esp],edi。然后发生了槽轮转。

![](https://bbs.kanxue.com/upload/attach/202610/960900_FDGRRE6MPQ9ZPEJ.webp) 

  继续跟踪，我们发现了 0x43367B 这个地址是 VMRet 后调用完. text 段或者系统 API 时的返回地址，即下一个 VMKey/VMEntry 入口。0x408F50 这个地址是 VMRet 后的目标地址，即退出 VM 时调用的函数。然后还有关于 flags 的 MBA 混淆，实际上这条指令没有带来任何的变化，可化简为 flags=falgs。因为 VMP 借此保护 JCC 控制流的还原，化简 MBA 过后。同时还注意到; slot#10 这个槽位固定为 0，所以说是死代码，可消除，是 VM 的固定模式在特定情况开启加密。（来自于 VMEntry 的 mov edx,0; push edx）

![](https://bbs.kanxue.com/upload/attach/202610/960900_HQU9JB399TN4GRN.webp) 

  经过这次的分析，我们成功还原了第一个 VM 的行为：不过值得注意的是，还原后的汇编无法直接替换第一个 VM 函数，因为第二个 VM 函数有校验。需要同时更改 vEIP 为第二个 VM 函数所需要的 Opcode 首地址才行。

![](https://bbs.kanxue.com/upload/attach/202610/960900_VBH5S5RHDFH8P48.webp) 

  VMP 关于 eflags 会利用 Nor 门实现个 MBA 混淆（左上角图）一般值无变化。如果是循环结构 vJmp 的 Opcode 地址参数来源从当前 vEIP 取一个 Const Imm 立即数转变为依赖 vPushImm4 传进去，传俩个，一个是 taken 目标地址一个是 fail 失败时地址，往回跳转的便是循环结构，反之这是普通的条件分支。（在分析循环的时候忘记条件跳转时产生的错误结论：一个是 Opcode 头部、一个是 Opcode 尾部。）如果判断要跳出循环则依赖传入一个新的 handler 地址与 0x815 之类的掩码计算。如果是寄存器轮转，则直接传入下一条 handler 地址。

  大概就是 vJmp 的参数有俩个，分别为 Opcode 地址、NextHandler。如果是循环，NextHandler 就直接在编译期算好，从 vEIP 取出来就行了但需要传 Opcode 地址。如果不是循环，Opcode 地址就在编译期算好，需要传个 NextHandler。

![](https://bbs.kanxue.com/upload/attach/202610/960900_NZ3VBTWQGGQU236.webp) 

指令模式与 Handler 分配机制表

<table><tbody><tr><td><p><strong>模式</strong></p></td><td><p><strong>Opcode 地址</strong></p></td><td><p><strong>NextHandler</strong></p></td><td><p><strong>典型场景</strong></p></td></tr><tr><td><p>（错误）循环 back-edge</p></td><td><p>运行时压栈传入（vPushImm4）</p></td><td><p>编译期定，从 vEIP 取立即数</p></td><td><p>循环体末尾跳回循环头</p></td></tr><tr><td><p>条件跳转</p></td><td><p>运行时压栈传入（vPushImm4）</p></td><td><p>编译期定，从 vEIP 取立即数</p></td><td><p>条件跳转</p></td></tr><tr><td><p>顺序流</p></td><td><p>编译期定，从 vEIP 取立即数</p></td><td><p>运行时压栈传入</p></td><td><p>普通指令顺序执行</p></td></tr><tr><td><p>寄存器轮转</p></td><td><p>编译期定</p></td><td><p>编译期定（直接传下一 handler 地址）</p></td><td><p>handler 轮换，无需运行时解析</p></td></tr></tbody></table>

  我们忽略了 vPush、vPop 前缀算子对栈的副作用，即 push 在 32bit 下会让 esp-4、而 pop 会让 esp+4。如果不做修复，我们将无法还原函数的参数传递等一系列基于栈 esp 的实现内容。

修复二级指令 vPushImm4、vPushReg4、vPushVEsp、vPopReg4 为基于一级指令 sub add 的表达

    call void asm sideeffect "; [13] vPushImm4 : push imm(0x1)", ""()

    call void @vPushImm4(i32 1)

    call void asm sideeffect "; [14] vPushReg4 : push slot#2[0x1C] = eax", ""()

    call void @vPushReg4(i32 2)

    call void asm sideeffect "; [17] vPopReg4 : pop -> slot#11[----] = flags", ""()

    call void @vPopReg4(i32 11)

    call void asm sideeffect "; [20] vPushVEsp : push vESP (esi)", ""()

    call void @vPushVEsp()

![](https://bbs.kanxue.com/upload/attach/202610/960900_PXVA6Q79EGHSQQM.webp) 

  由于 VMProtect 基于栈实现了 MBA 混淆，利用 GAMBA 解混淆同时用 z3 验证正确性，这里给出 GAMBA 的适配方案。

降级策略：GAMBA 表达不了或解析不了的算子（mem/shld/shrd/idiv/mul/sub/pow /shr/shl）先递归化简其参数，再整体登记为一个不透明变量。这样外围 MBA 仍能被化简，且语义严格等价（变量化不改变任何东西，只是让 GAMBA 看不见内部）。

GAMBA 的 simplify() 内部每次化简都起一个 multiprocessing.Process + Manager，因此内联内部调用函数进行优化：

<table><tbody><tr><td><p>方式</p></td><td><p>285 次化简耗时</p></td></tr><tr><td><p>simplify_mba()（原 API）</p></td><td><p>166 秒</p></td></tr><tr><td><p>gamba_simplify_fast()（内联三步）</p></td><td><p>0.8 秒</p></td></tr></tbody></table>

![](https://bbs.kanxue.com/upload/attach/202610/960900_NXUASYH6T6F5QGF.webp) 

  在降级二级指令 push、pop 为基于一级指令 sub add 的表达，以及其他 handler 关于 esp 变化都正确的基础上，我们可以针对 VMProtect 栈机产生的变量进行活跃分析，前提是将栈帧 SSA 化。在死代码消除的基础上，我们根据栈机的特性，指定一个规则 PUSH、AUL（位运算）不生成指令，只在栈上记值；直到 pop 到寄存器 / 写内存 / 控制流边界 (vJmp) 时，才把整条表达式折叠成以目标寄存器为目的的指令序列。这样 push 0xFFFFF7EA 会变成 and eax, 0xFFFFF7EA 里的立即数操作数。同时需要注意如何防止死代码消除删掉栈里的参数，如下方第三张图所示。

![](https://bbs.kanxue.com/upload/attach/202610/960900_4KUSAR43UMZN8T3.webp) 

![](https://bbs.kanxue.com/upload/attach/202610/960900_D9AASE5FJ2R3Q72.webp)

![](https://bbs.kanxue.com/upload/attach/202610/960900_RUNJ4EK3298ACY6.webp) 

  vm2asm_ssa/isel 把栈机表达式在 pop、写内存、vJmp、vRet 这些消费点物化成伪汇编，并做死值消除；控制流在 vm2cfg 里按 vJmp/vRet 切段，从 target=(A0&mask)|(A1&~mask) 的掩码选择模式识别条件跳转、用候选 VM 地址的引用次数找汇合点 / 循环头，生成 cfg_graph.json/DOT；denoise 再把. jcc 语义化并保守切片去混淆；最后 asm2ir 把每个 CFG 节点映射成一个 LLVMBasicBlock，verified 的. jcc 变成真实 icmp 分支（否则回退 flags&bit），用 vstk/vctx 数组和 @g_ds 表达 VM 栈与 guest 内存，llc 出 obj。

![](https://bbs.kanxue.com/upload/attach/202610/960900_HWHY6HSXUPX48RH.webp) 

![](https://bbs.kanxue.com/upload/attach/202610/960900_XDNPEF3M4W67KWV.webp) 

  现在我们具备了还原 VM 的能力了，由于我们借鉴了 xxvm 的经验，所以用他的 demo（https://bbs.kanxue.com/thread-247040.htm/ [原创] 深入解析 VMP3.x_VM-Jcc 指令爆破 ）进行测试，后续再还原我们相对复杂的古典加密算法 CrackMe。第一个函数 0052F9FD 如下图所示，IDA 参数顺序反了，原因是 2LLVMIR 对函数参数的解析顺序反了，这就导致了显示异常。第一个 VM 函数为读取 serial 内容，这里输入了 “123，123“

![](https://bbs.kanxue.com/upload/attach/202610/960900_3BG89HC8M2RK8PG.webp) 

  执行完这个 VM，到下一个 VM 函数入口 004C7704 下断，此时 Dbg 的寄存器窗口 EAX 已经变为了 “123123”，我们右键插件 malware->vmp3.x-> 还原 VM，只需要填写偏移 n 为 44（VMP 初始化时候所依赖的 11 个槽）然后点击开始还原即可，默认使用的是 xxvm 的静态模拟执行模式，如果取消勾选则使用 unicorn 进行模拟。

![](https://bbs.kanxue.com/upload/attach/202610/960900_CAJ74JXKBHUHG7D.webp) 

  第二个 VM 函数的执行内容为字符串比较，比较用户输入的 serial 是否为空，其中 IDA 反编译的结果存在这样的问题 flags 在 VM 调度时的位置，因没有完全消除 VM 调度的噪音并标记为 eflags，导致无法理解（也有可能是寄存器）。

![](https://bbs.kanxue.com/upload/attach/202610/960900_5A82RU9U6S6BFA5.webp) 

  来到第三个 VM 函数 004C2E16，这里出现了第一个条件跳转，同时 IDA 反编译结果存在一定的问题，条件跳转应当还原为二级 JZ 指令而非基于一级指令 eflags&mask 的条件判断，当输入 serial 不为空时，调用函数 00401261，下一个 VM 入口，当输入 serial 为空时，弹窗没有输入密码，不是退出虚拟机，因为 xxvm 只实现了 trace 一条路径，只能分析这一条路径的 VM 功能。

  注意到右上角方框部分，我们提供了一个 JCC 即 vJmp 的地址和俩个参数 Taken、Fail 正确和失败时的 OpCode 地址，以及最后一个计算出 eflags 的 handler 地址，可以看到 vAdd4，我们可以对其产生的标志位进行修改测试下。

![](https://bbs.kanxue.com/upload/attach/202610/960900_ZSV5JUZH6M7P9EE.webp) 

  我们给 vJmp 溯源，可以看到最后一个 ALU 位运算的 handler 地址，在栈里将 flags&mask 的结果取反置为 1，可以看到程序成功运行 “请输入密码”。

![](https://bbs.kanxue.com/upload/attach/202610/960900_FRNZNEJ7FHTPXEQ.webp) 

![](https://bbs.kanxue.com/upload/attach/202610/960900_UNPBP44J9Z8K997.webp) 

  同时可以看到该 Handler 最终执行只执行 1 次，这说明我们如果要 Patch 这个程序只需要给 004D0CC9 这个地址挂 hook，给 flag&mask 的结果强制置为 0 即可。接下去我们开始分析第四个 VM 函数 00521A68，内容是读取编辑框内容，第五个 VM 函数地址是 531C79。

![](https://bbs.kanxue.com/upload/attach/202610/960900_RHU7WJG84ZSNSC8.webp) 

在第五个 VM 函数 531C79 地址下断点，看汇编，程序行为是调用字符串比较函数，传入 CrAcKingSeven7 和用户输入的 “123123” 进行比较。

![](https://bbs.kanxue.com/upload/attach/202610/960900_BJV44EB275ER8YY.webp) 

  在第六个 VM 函数 004D890F 地址下断点，可以看到有个条件跳转，在最后一个产生 eflags 的 ALU 位运算 handler 对产生的 flags&mask 置为 1，代表着我们输入的内容与预期的字符串相同。当然，如果你知道另一个 Opcode 的地址，可以直接在 vJmp 挂 Hook 进行操作，把 [vSP] 直接改成 taken/fail，实现 vEIP 的劫持（更为稳定但缓慢的方式是溯源产生影响跳转的 eflags 的 ALU 指令，对 zf 位取反）。

可以看到，程序弹窗，“哇，大牛逼！恭喜你，成功啦！”。

![](https://bbs.kanxue.com/upload/attach/202610/960900_B89QK4XWE6U864W.webp) 

  整理一下程序逻辑应当是这样的，至此，我们还原了 xxvm 的 demo 程序。

![](https://bbs.kanxue.com/upload/attach/202610/960900_3VUU7ZWMQWP4FHD.webp) 

  如果需要还原程序被 VM 函数的完整逻辑，就需要解决 xxvm 本身只模拟一条 Trace 路径导致 CFG 不完整的问题。根据上述手动修改 VM 中条件跳转的方法，先模拟一条 Trace，识别出条件跳转结构，知道另一个 Opcode 的地址，保存快照，在 vJmp 挂 Hook 进行操作把 [vSP] 直接改成 taken/fail，实现 vEIP 的劫持。更为稳定但缓慢的方式是溯源产生影响跳转的 eflags 的 ALU 指令，对 zf 位进行取反。最终模拟出另一条分支。然后循环往复，做好去重即可拿到程序被 VM 的全部分支。但需要注意的是，这仅是模拟一个局部 VM 的全部分支，在 vRet 后需要在同步 vEIP 的基础上进一步地模拟下一个 VM 入口才能逐步还原整个 VM 函数的控制流。

![](https://bbs.kanxue.com/upload/attach/202610/960900_QE99J7W7PEUETPD.webp) 

![](https://bbs.kanxue.com/upload/attach/202610/960900_XAZMR24JKAMDHJD.webp) 

  在栈机模拟工具针对槽轮转进行还原产生了槽和物理寄存器的轮转映射表，同时我们利用 GAMBA 化简了大量的 MBA 混淆，虽然程序当中依然存在一些不透明的变量导致 MBA 混淆没有化简。并利用伪汇编将其转为 LLVMIR。

  在这样的条件下，我们可以基于 LLVM 的 alloca 特性让线性的 LLVMIR 得以不断地常量折叠与化简，最终移除掉 VMProtect 的寄存器轮转结构。

  32 位程序一般通过还原对原始程序 stack 操作做活跃变量分析，通常对 stack 先 read 而不 write 的区域就是参数了。这里对 IR 编写一个 pass 进行处理，最终还原出第二个 VM 函数 0x0043367B 的参数为 2 个。因为只有局部 VM，所以必须再添加一个 vminit。

![](https://bbs.kanxue.com/upload/attach/202610/960900_5HWP5M84WHD63ED.webp) 

  当退出局部 VM 的时候，会调用目标函数，如果不在最后一个 VM 里，那么我们需要对__vmp_return 进行一下处理，还原出参数。根据 vRet 排除 8 个必然出现还原的物理寄存器，追踪真正对程序栈进行写入后没有再次读取与写入的情况同时要保证与栈中关于中间运算产生的死代码相切割。下图为第二个局部 VM 函数结束时的 API 调用。

![](https://bbs.kanxue.com/upload/attach/202610/960900_6EVQJ5MZSB29WGT.webp) 

  由于 VMProtect 针对一个函数当中存在多个其他函数调用的情况会采用退出虚拟机的方式使得目标函数能够调用（如果其他函数也被 VM 了，那该函数将会直接内联到 VM 里执行而不退出虚拟机）

  这个机制使得程序会产生一些分散的局部 VM，如果要完整分析一个函数的行为则需要对其进行合并。有多少个局部 VM 可以根据有多少 vRet 进行溯源推测，在我们还原一个局部 VM 函数产生 obj 时，可以对产生的__vmp_return 符号进行追踪分析其参数，第一个参数为目标函数地址，第二个参数为下一个 VM 函数入口。根据下一个 VM 函数入口继续追踪，虽然下一个 VM 函数入口的参数依赖目标函数的返回值，但是不继续追踪，连完整的函数 CFG 都还原不了。基于此，可以不断地溯源追踪，追踪将局部 VM 连成一条链。

  现在我们合并了局部的 VM，开始给第一个函数进行分析，可以看到该程序调用了 strlen、strcpy 参数。结合程序给的 “古典加密算法 Crackme” 信息，我们可以构造这样的提示词“分析算法，该算法以去除了大量的虚拟化，从古典加密算法的置换、仿射、多表仿射角度进行分析”

![](https://bbs.kanxue.com/upload/attach/202610/960900_EJ5YZ26XSYUUNVK.webp) 

  根据 DeepSeek 的分析，第一个函数大概率是列置换和路由置换。

![](https://bbs.kanxue.com/upload/attach/202610/960900_J5S3CGRTGQ7HKSE.webp) 

  第二个函数 0x0041BE0F 存在递归调用的情况，在之前的上下文基础上。给全部函数作为上下文丢进去，“这是置换、仿射还是多表仿射”。根据 Deepseek 的分析，第二个函数大概率是仿射变换。

![](https://bbs.kanxue.com/upload/attach/202610/960900_C679S8BWXYDHP8R.webp) 

![](https://bbs.kanxue.com/upload/attach/202610/960900_U8GXX9CAX49PGXM.webp) 

  2026 年 10 月 23 日，由于时间不多了，只好在此停下。人工智能的发展很迅速，遇到非预期的情况利用 agent 开个 workflow 去排查修复，源码可以看文章附件。不过由于都是 DeepSeek 写的，有很多 python 脚本，印象里某次睡前开了个 workflow 去排查个全部分支模拟的错误，为了一个异常写了一堆内容，导致效率慢了几十倍。后续如果要继续开发工具，应该会依据本文章的内容让 DeepSeek 严格实现。

仓库：https://github.com/llvm/llvm-project

能力：一套模块化、可重用的编译器与工具链基础设施，核心是 LLVM IR——一种带类型的、SSA 形式的中间表示。前端（Clang 等）把源码降到 IR，后端把 IR 降到目标机器码，中间夹着一整套与目标无关的优化 pass（常量折叠、GVN、死代码消除、指令合并等）。对本工作而言，LLVM 的价值有两层：一是它提供了 IR 层面的规范化与优化框架，很多 MBA 混淆在 IR 上会先被部分折叠；二是它的 pass 基础设施和 opt/llc 工具链，方便把提升后的代码送进自定义分析或化简流程。

作者：Trail of Bits（成立于 2012 年，总部位于美国纽约的一家独立的安全研究与咨询公司）

仓库：https://github.com/lifting-bits/remill

能力：一个二进制提升（lifting）库，把 x86、x86-64、AArch64 等架构的机器指令语义翻译成 LLVM IR。它按指令逐条建模语义（每条指令对应一段 IR 片段），输出的是可被 LLVM 工具链继续处理的 IR 模块。在「二进制→IR →化简→反编译 / 分析」这条流水线里，Remill 是入口：把目标二进制提升成 LLVM IR，后续的优化、MBA 识别与化简都在 IR 层完成。

近期有报道称 OpenAI 正在就收购 Trail of Bits 进行初步谈判。

OpenAI in early talks to buy Trail of Bits, The Information reports

https://thenextweb.com/news/openai-trail-of-bits-acquisition-talks

作者：Tim Blazytko（Automated Detection of MBA Obfuscation 系列文章作者）

仓库：https://github.com/DenuvoSoftwareSolutions/SiMBA（Rust 实现，也有 Python 绑定思路）

能力：基于线性 MBA 理论（Σ cᵢ·xᵢ + Σ cᵢⱼ·xᵢ∧xⱼ + ...），把表达式规约成线性组合，再查表合成最简形式。对 nor/nand/and/or/xor + add/sub + not 这类位运算 + 加法混合几乎是量身定做。

局限：主要覆盖线性 MBA（bitwise 与算术的线性组合）；shl/shr/shld/shrd/idiv 不在核心覆盖内——但你的表达式里这些多半是外围结构，MBA 核心在 nor/add 层，可以剥离处理。

https://github.com/DenuvoSoftwareSolutions/GAMBA

关键信息

归属：Denuvo Software Solutions（与 SiMBA 同一团队）。

定位：基于 SiMBA 迭代化简非线性 MBA 表达式，核心思路是隔离线性子式、重构后交给 SiMBA 处理。

主程序：simplify_general.py 用于通用 MBA，simplify.py 用于线性 MBA。

验证选项：加 -z 可调用 Z3 做等价性验证，确保化简结果语义正确。

与你日志的匹配点：你之前担心的 shl/shr/idiv 与位运算混叠问题，正好是 GAMBA“子式变量替换 + 迭代 SiMBA” 策略试图覆盖的场景。命令行直接传表达式即可，格式简单：

bash

python3 src/simplify_general.py "你的表达式"

AVLCode

作者 / 团队：安天澜砥团队（安天科技集团）

仓库 / 官网：https://www.avlcode.cn

能力：一款国产 AI 智能编程与安全分析桌面助手，核心定位为 “智能编码开发 + 二进制安全分析” 双主线。底层搭载安天自研的澜砥威胁检测分析垂直大模型，支持亿级 Token 超长上下文，原生面向二进制数据处理。安全分析侧内置一套只读工具集，包括哈希与熵计算、字符串与 IOC 抽取、PE/ELF/Mach-O 解析、反汇编、C/Java 伪代码反编译、YARA 规则匹配，并明确提及 “代码反混淆与数据解密” 能力。全链路适配国密 SM2/SM3/SM4，样本与业务数据在本地部署模式下不出域，支持 Windows/macOS/Linux 及国产操作系统原生分发。

https://bbs.kanxue.com/homepage-674721.htm

https://bbs.kanxue.com/thread-250711-1.htm

VMProtect xx_vm Chinese Decompiler

xx_vm_releasevmp 虚拟代码分析还原插件，可以自动化的分析由 vmp 保护的虚拟代码，提供了包含丰富信息的日志文件，通过对日志伪代码的分析，我们几乎可以还原出原始指令。从而达到分析目的。目前我们已经开发出 ollydbg 适用的插件，后续我们也会对接其他主流调试器。

Support Version 3.x

Support Debuger: ollydebug, x64dbg

感谢看雪论坛，这是一片净土……

感谢安天，这是一座泰山北斗……

还有 JMPOEP、飘云阁、吾爱破解、学破解论坛、精易论坛，这是属于内地在世界上所独有的文化

  这篇工具与文章是帮工厂客户解决某激光排版软件加密狗时匆忙的制作，很多内容相对不完善，例如在 VM 参数还原部分在局部 VM 函数最终合并时会失灵。程序关于局部变量的读写没有移除 vESP 的结构，而是将 [vESP+n] 的读写抬升为一个变量，这就产生了 byte_18FF00 与 byte_18FF12 这样的情况，对于数组不够友好。同时在控制流分支边界条件的还原不够充分，只能还原出程序控制流的基本骨架，甚至可能把 for 循环还原成递归的形态，对于加壳的 VMProect 保护程序需先脱壳，总之当前还存在很多的问题…… 如果未来业务还在发展，应该会继续开发这款工具。

文章基于 remill 的通用混淆削弱工具暂时还没实现好，只是利用 AVLCode 验证了可行性，我们可以快速地基于澜砥 VILLM 等专门面向二进制汇编分析的大模型分析程序当中存在的混淆规律，然后利用 remill 将混淆的二进制程序翻译为 LLVM 的 IR，将澜砥大模型分析的混淆规律交由开发 Agent 例如 Cursor、ClaudeCode 快速开发 LLVM 的 pass 验证混淆削弱的可行性。

  文章关于 VM 分析的部分主要基于论坛 xxvm 作者开发的工具进行二次开发实现，在分析样本的过程中我们逐步拆解了 VMProtect 的架构，并尝试给出了一定的还原方案，最终作者还原了 xxvm 的 demo 程序以及分析了古典加密算法 CrackMe 的行为，但关于 vstk 和 vJmp 条件跳转的还原部分依然不够理想。

近期有报道称是 OpenAI 正在就收购 Trail of Bits 进行初步谈判，显然，基于 LLVM 和二进制翻译的 Cyber 大模型安全将成为一种重要的方向：The two companies have worked together since June. They launched Patch the Planet, a project inside OpenAI’s Daybreak cyber programme. Trail of Bits engineers use OpenAI’s GPT-5.5-Cyber model and its Codex tools to find bugs in open-source projects. They check the findings, write patches and work with the maintainers on fixes.

  OpenAI is trying to build a substantial cybersecurity business, The Information reported. In September, it put $1bn behind cyber defence for water utilities and community banks under Daybreak.

  OpenAI in early talks to buy Trail of Bits, The Information reports

https://thenextweb.com/news/openai-trail-of-bits-acquisition-talks

![](https://bbs.kanxue.com/upload/attach/202610/960900_SFYGHJX7VG6SBSH.webp) 

[[招生] 科锐逆向工程师培训 (2026 年 11 月 13 日实地，远程教学同时开班, 第 57 期)](https://bbs.kanxue.com/thread-51839.htm)

最后于 1 小时前 被 zZhouQing 编辑 ，原因： 上传附件