> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [valsamaras.medium.com](https://valsamaras.medium.com/arm-64-assembly-series-basic-definitions-and-registers-ec8cc1334e40)

> Main Definitions

**ARM** is the acronym for **Advanced** **RISC** **Machines** and if it is not followed by a noun, it refers to a family of processors (CPUs) that are designed based on the architecture developed by [**Arm Ltd.**](https://en.wikipedia.org/wiki/Arm_(company))**,** a British company based in Cambridge, England. The **RISC** is another acronym which stands for **Reduced Instruction Set Computer** and comes as an alternative to **CISC** which stands for **Complex** **Instruction Set Computer,** used by Intel processors. You may find their main differences bellow, but for the purpose of this workshop, just keep in mind the **reduced instruction set** (as the term implies) as well as the larger number of general purpose registers of the RISC based machines.

![](https://miro.medium.com/max/1400/1*UrKmzIxvr0R3KbKsYt0sIw.png)

CISC vs RISC source: [https://www.microcontrollertips.com/risc-vs-cisc-architectures-one-better/](https://www.microcontrollertips.com/risc-vs-cisc-architectures-one-better/)

Following up with **arm versioning related terms** is rather overwhelming, but the most important thing to remember is that besides **Arm Ltd**., third party vendors are also producing CPUs based on this architecture, including **Apple**, **Nvidia**, **Qualcomm**, **Samsung** etc. Each family of processor supports a specific instruction set which corresponds to an ARM architecture version. So we have `**ARMv1, ARMv2,…,ARMv8**` etc. with `[**ARMv9**](https://www.arm.com/products/silicon-ip-cpu)` being the most recent at the time that this post is written. `**Cortex-A**` from Arm Ltd., Qualcomm’s `**Snapdragon**`**,** Samsung’s `**Exynos**`**,** Apple’s `**Mx**` are some examples of CPU families designed to support the latest architecture versions. You can find an updated list [here](https://en.wikipedia.org/wiki/List_of_ARM_processors).

From a programmers point of view, a term that you might find useful is the **instruction set architecture** (**ISA**) which defines the set of instructions and the set of registers that the hardware must support. Being familiar with the ISA is the first step in learning the arm assembly language. **In this series of posts we are going to focus on** `**64-bit ISA for AArch64**` **processors.**

The `AArch64 ISA`defines **31** **general purpose registers** which can be used for 64- or 32-bit values and you may find them referenced either as `**Xn, Wn** or **Rn** (**capitalisation is optional**)**, where n is the index of register (a number from 0 to 30).**`

*   When used for 64 bits they are referenced with the letter **X**
*   When used for 32 bits they are referenced with the letter **W**
*   or more generally (size irrelevant) you may find them referenced with the letter **R**

For example, **to refer to the lower part of the register with 0 index** use the symbol `**W0**` and for all the 64 bits you may use `**X0**` **:**

![](https://miro.medium.com/max/1118/1*7afD5Y42ygGPdxPGL-gwmg.png)

Source: [ARMv8_InstructionSetOverview.pdf](http://harmanani.github.io/classes/csc320/Notes/ch02.pdf)

Another acronym you mind find useful is the _application binary interface —_ **ABI** which defines how these registers are used. So, this **conventional** rather than **constructional** related usage, defines the following rules:

*   **Registers R0 to R7** are used to save arguments when calling a function (see below), while the R0 is used also to store the result which is returned by a function.

> *[Note](https://valsamaras.medium.com/introduction-to-x64-linux-binary-exploitation-part-1-14ad4a27aeef): When a function is called, the compiler uses, a **stack frame** (allocated within the program’s runtime **stack**) in order store all the temporary information that the function requires to operate. Depending on the **calling convention** the caller function will place the aforementioned information in specific registers or in the program stack or in both. For example for the **_C calling convention_** _(cdecl)_ on a Linux operating system, up to six arguments will be placed to the RDI, RSI, RDX, RCX, R8 and R9 registers and anything additional will be placed in to the stack. Let’s see a simple example, in order to understand this.

*   **Register R8** (Indirect result location register), used in C++ for returning non-trivial objects (set by the caller).
*   **Registers R9 to R15** (known as **scratch registers**) can be used any time without any assumptions about their contents.
*   **Registers R16, R17 (**intra-procedure-call temporary registers) the linker may use these in PLT code. Can be used as temporary registers between calls.
*   **Register R18** (platform register) reserved for the use of platform ABI. For example, for the windows ABI, in kernel mode, points to [KPCR](https://en.wikipedia.org/wiki/Processor_Control_Region) for the current processor; in user mode, points to [TEB](https://docs.microsoft.com/en-us/windows/win32/debug/thread-environment-block--debugging-notes-).
*   **Registers R19-R28** can also be used as scratch registers, but their contents must be saved before usage and restored afterwards.
*   **Register** **R29** is used as a `**Frame Pointer**`, pointing to the current stack frame and it is usually useful when the program runs under a debugger. The GNU C compiler has an option to use **R29** as a general purpose register via the `-fomit-frame-pointer` option.
*   **Register** **R30** is known as the **link register (lr)** can be used to store the return address during a function call, an alternative of saving the address to the call stack. Certain branch and link instructions store the current address to the link register before the program counter (the register that holds the address of the next instruction) loads the new address.
*   The **stack pointer (sp)** stores the address corresponding to the end of the stack (or as commonly said pointing to the top of the stack). This address will change when registers are pushed into the stack or during the allocation or deletion of local variables.
*   The **program counter (pc)** holds the next address which contains the code to be executed. It is increased automatically by four after each instruction while it can be modified by a small number of instructions (e.g. adr or ldr). This gives the ability of jumping to a new address and start executing code from there.
*   The **zero register** (referenced as **zr**, **xzr** or **wzr**)is a [special](https://en.wikichip.org/w/index.php?title=special-purpose_register&action=edit&redlink=1) [register](https://en.wikichip.org/wiki/register) that is hard-wired to the integer value `0`. Writing to that register is always discarded and reading its value will always result in a `0` being read:

![](https://miro.medium.com/max/1400/1*CpT9ukirZHek24DoA4dx5g.png)

Source: [ARMv8_InstructionSetOverview.pdf](http://harmanani.github.io/classes/csc320/Notes/ch02.pdf)

*   The **PSTATE** **register** is a collection of fields which are mostly used by the OS. The user programs make use of the first four bits, which are marked as N,Z,C and V respectively (referred as _condition flags_) where each one of them is interpreted as follows:

N → **Negative** , the flag is set to 1 when the (signed) result of an operation is negative.

Z → **Zero,** the flag is set to 1 when the result of an operation is 0 and set to 0 when the result is not 0.

C →**Carry,** the flag is set to 1 if an add operation results in a carry or a subtraction operation results in a borrow.

O →**Overflow,** set to 1 if a signed overflow occurs during an addition or subtraction.

![](https://miro.medium.com/max/1400/1*6UJmLzfYnOOZbtAJGUS4Vg.png)

The PSTATE register

**The AArch64 Registers of an Apple Silicon Macs (M1):**

![](https://miro.medium.com/max/1400/1*Fs-PmlRPoMIJ22p0737F9Q.png)

The AArch64 architecture also supports 32 floating-point/[SIMD](https://en.wikipedia.org/wiki/Single_instruction,_multiple_data) registers, numbered from 0 to 31, where each can be accessed as a full **128** bit value (using v0 to v31 or q0 to q31), 64-bit value (using d0 to d31), 32-bit value (using s0 to s31) as a 16-bit value (using h0 to h31), or as an 8-bit value (using b0 to b31). Accesses smaller than 128 bits only access the lower bits of the full 128-bit register. **They leave the remaining bits untouched unless otherwise specified.**

*   **ARM** is the acronym for **Advanced** **RISC** **Machines**
*   **RISC, used by arm processors** stands for **Reduced Instruction Set Computer** and comes as an alternative to **CISC** which stands for **Complex** **Instruction Set Computer,** used by Intel processors.
*   The **instruction set architecture** (**ISA**) defines the set of instructions and the set of registers that the hardware must support.
*   The 64bit ISA for AArch64 (AArch64 ISA) defines **31** **general purpose 64 bit registers.**
*   The application binary interface _—_ **ABI** is a convention on how the registers are used.
*   The **Program Counter** and **Stack Pointer** are no longer general purpose and therefore cannot be used by most instructions
*   The AArch64 architecture also supports **32 floating-point/**[**SIMD**](https://en.wikipedia.org/wiki/Single_instruction,_multiple_data) **registers**, numbered from 0 to 31 that can be accessed in different sizes: 8, 16, 32, 64 and 128 bits.