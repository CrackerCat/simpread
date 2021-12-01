> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [secret.club](https://secret.club/2021/09/08/vmprotect-llvm-lifting-1.html)

> This series of posts delves into a collection of experiments I did in the past while playing around w......

This series of posts delves into a collection of experiments I did in the past while playing around with LLVM and VMProtect. I recently decided to dust off the code, organize it a bit better and attempt to share some knowledge in such a way that could be helpful to others. The macro topics are divided as follows:

*   Part 1: Lifting
*   [Part 2](https://secret.club/2021/09/08/vmprotect-llvm-lifting-2.html): Exploration
*   [Part 3](https://secret.club/2021/09/08/vmprotect-llvm-lifting-3.html): Optimization

First, let me list some important events that led to my curiosity for reversing obfuscation solutions and attack them with LLVM.

*   In 2017, a group of friends (SmilingWolf, mrexodia and xSRTsect) and I, hacked up a Python-based devirtualizer and solved a couple of [VMProtect challenges](https://forum.tuts4you.com/topic/39481-devirtualizeme-vmprotect-309/) posted on the Tuts4You forum. That was my first experience reversing a known commercial protector, and taught me that writing compiler-like optimizations, especially built on top of a not so well-designed IR, can be an awful adventure.
*   In 2018, a person nicknamed RYDB3RG, posted on Tuts4You a [first insight](https://forum.tuts4you.com/topic/40832-vmprotect-vs-llvm/) on how LLVM optimizations could be beneficial when optimising VMProtected code. Although the easy example that was provided left me with a lot of questions on whether that approach would have been hassle-free or not.
*   In 2019, at the SPRO conference in London, [Peter](https://twitter.com/Blips_and_Chitz) and I presented a [paper](https://arxiv.org/abs/1909.01752) titled “SATURN - Software Deobfuscation Framework Based On LLVM”, proposing, you guessed it, a software deobfuscation framework based on LLVM and describing the related pros/cons.

The ideas documented in this post come from insights obtained during the aforementioned research efforts, fine-tuned specifically to get a good-enough output prior to the recompilation/decompilation phases, and should be considered as stable as a proof-of-concept can be.

Before anyone starts a war about which framework is better for the job, pause a few seconds and search the truth deep inside you: every framework has pros/cons and everything boils down to choosing which framework to get mad at when something doesn’t work. I personally decided to get mad at LLVM, which over time proved to be a good research playground, rich with useful analysis and optimizations, consistently maintained, sporting a nice community and deeply entangled with the academic and industry worlds.

With that said, it’s crystal clear that LLVM is not born as a software deobfuscation framework, so scratching your head for hours, diving into its internals and bending them to your needs is a minimum requirement to achieve your goals.

I apologize in advance for the ample presence of long-ish code snippets, but I wanted the reader to have the relevant C++ or LLVM-IR code under their nose while discussing it.

The following diagram shows a high-level overview of all the actions and components described in the upcoming sections. The blue blocks represent the inputs, the yellow blocks the actions, the white blocks the intermediate information and the purple block the output.

![](https://secret.club/assets/vmp2llvm/diagram2.svg)

Enough words or code have been spent by others ([1](https://www.msreverseengineering.com/blog/2014/6/23/vmprotect-part-0-basics), [2](https://webtv.univ-lille.fr/video/7566/inside-vmprotect), [3](https://github.com/can1357/NoVmp), [4](https://back.engineering/17/05/2021/)) describing the virtual machine architecture used by VMProtect, so the next paragraph will quickly sum up the involved data structures with an eye on some details that will be fundamental to make LLVM’s job easier. To further simplify the explanation, the following paragraphs will assume the handling of x64 code virtualized by VMProtect 3.x. Drawing a parallel with x86 is trivial.

[Liveness and aliasing information](#liveness-and-aliasing-information)
-----------------------------------------------------------------------

Let’s start by saying that many deobfuscation tools are completely disregarding, or at best unsoundly handling, any information related to the [aliasing](https://en.wikipedia.org/wiki/Aliasing_(computing)) properties bound to the memory accesses present in the code under analysis. LLVM on the contrary is a framework that bases a lot of its optimization passes on precise aliasing information, in such a way that the semantic correctness of the code is preserved. Additionally LLVM also has strong optimization passes benefiting from precise [liveness](https://en.wikipedia.org/wiki/Live_variable_analysis) information, that we absolutely want to take advantage of to clean any unnecessary stores to memory that are irrelevant after the execution of the virtualized code.

This means that we need to pause for a moment to think about the properties of the data structures involved in the code that we are going to lift, keeping in mind how they may alias with each other, for how long we need them to hold their values and if there are safe assumptions that we can feed to LLVM to obtain the best possible result.

A suboptimal representation of the data structures is most likely going to lead to suboptimal lifted code because the LLVM’s optimizations are going to be hindered by the lack of information, erring on the safe side to keep the code semantically correct. Way worse though, is the case where an unsound assumption is going to lead to lifted code that is semantically incorrect.

At a high level we can summarize the data-related virtual machine components as follows:

*   **30 virtual registers**: used internally by the virtual machine. Their liveness scope starts after the `VmEnter`, when they are initialized with the incoming host execution context, and ends before the `VmExit(s)`, when their values are copied to the outgoing host execution context. Therefore their state should not persist outside the virtualized code. They are allocated on the stack, in a memory chunk that can only be accessed by specific `VmHandlers` and is therefore guaranteed to be inaccessible by an arbitrary stack access executed by the virtualized code. They are independent from one another, so writing to one won’t affect the others. During the virtual execution they can be accessed as a whole or in subregisters. From now on referred to as `VmRegisters`.
    
*   **19 passing slots**: used by VMProtect to pass the execution state from one `VmBlock` to another. Their liveness starts at the epilogue of a `VmBlock` and ends at the prologue of the successor `VmBlock(s)`. They are allocated on the stack and, while alive, they are only accessed by the push/pop instructions at the epilogue/prologue of each `VmBlock`. They are independent from one another and always accessed as a whole stack slot. From now on referred to as `VmPassingSlots`.
    
*   **16 general purpose registers**: pushed to the stack during the `VmEnter`, loaded and manipulated by means of the `VmRegisters` and popped from the stack during the `VmExit(s)`, reflecting the changes made to them during the virtual execution. Their liveness scope starts before the `VmEnter` and ends after the `VmExit(s)`, so their state must persist after the execution of the virtualized code. They are independent from one another, so writing to one won’t affect the others. Contrarily to the `VmRegisters`, the general purpose registers are always accessed as a whole. The flags register is also treated as the general purpose registers liveness-wise, but it can be directly accessed by some `VmHandlers`.
    
*   **4 general purpose segments**: the `FS` and `GS` general purpose segment registers have their liveness scope matching with the general purpose registers and the underlying segments are guaranteed not to overlap with other memory regions (e.g. `SS`, `DS`). On the contrary, accesses to the `SS` and `DS` segments are not always guaranteed to be distinct with each other. The liveness of the `SS` and `DS` segments also matches with the general purpose registers. A little digression: in the past I noticed that some projects were lifting the stack with an intra-virtual function scope which, in my experience, may cause a number of problems if the virtualized code is not a function with a well-formed stack frame, but rather a shellcode that pops some value pushed prior to entering the virtual machine or pushes some value that needs to live after exiting the virtual machine.
    

[Helper functions](#helper-functions)
-------------------------------------

With the information gathered from the previous section, we can proceed with defining some basic LLVM-IR structures that will then be used to lift the individual `VmHandlers`, `VmBlocks` and `VmFunctions`.

When I first started with LLVM, my approach to generate the needed structures or instruction chains was through the [IRBuilder](https://llvm.org/doxygen/classllvm_1_1IRBuilder.html) class, but I quickly realized that I was spending more time looking at the documentation to generate the required types and instructions than actually focusing on designing them. Then, while working on SATURN, it became obvious that following [Remill](https://github.com/lifting-bits/remill)’s approach is a winning strategy, at least for the initial high level design phase. In fact their idea is to implement the [structures](https://github.com/lifting-bits/remill/tree/master/include/remill/Arch/X86/Runtime) and [semantics](https://github.com/lifting-bits/remill/tree/master/lib/Arch/X86/Semantics) in C++, compile them to LLVM-IR and dynamically load the generated bitcode file to be used by the lifter.

Without further ado, the following is a minimal implementation of a stub function that we can use as a template to lift a `VmStub` (virtualized code between a `VmEnter` and one or more `VmExit(s)`):

```
struct VirtualRegister final {
  union {
    alignas(1) struct {
      uint8_t b0;
      uint8_t b1;
      uint8_t b2;
      uint8_t b3;
      uint8_t b4;
      uint8_t b5;
      uint8_t b6;
      uint8_t b7;
    } byte;
    alignas(2) struct {
      uint16_t w0;
      uint16_t w1;
      uint16_t w2;
      uint16_t w3;
    } word;
    alignas(4) struct {
      uint32_t d0;
      uint32_t d1;
    } dword;
    alignas(8) uint64_t qword;
  } __attribute__((packed));
} __attribute__((packed));

using rref = size_t &__restrict__;

extern "C" uint8_t RAM[0];
extern "C" uint8_t GS[0];
extern "C" uint8_t FS[0];

extern "C"
size_t HelperStub(
  rref rax, rref rbx, rref rcx,
  rref rdx, rref rsi, rref rdi,
  rref rbp, rref rsp, rref r8,
  rref r9, rref r10, rref r11,
  rref r12, rref r13, rref r14,
  rref r15, rref flags,
  size_t KEY_STUB, size_t RET_ADDR, size_t REL_ADDR,
  rref vsp, rref vip,
  VirtualRegister *__restrict__ vmregs,
  size_t *__restrict__ slots);

extern "C"
size_t HelperFunction(
  rref rax, rref rbx, rref rcx,
  rref rdx, rref rsi, rref rdi,
  rref rbp, rref rsp, rref r8,
  rref r9, rref r10, rref r11,
  rref r12, rref r13, rref r14,
  rref r15, rref flags,
  size_t KEY_STUB, size_t RET_ADDR, size_t REL_ADDR)
{
  // Allocate the temporary virtual registers
  VirtualRegister vmregs[30] = {0};
  // Allocate the temporary passing slots
  size_t slots[19] = {0};
  // Initialize the virtual registers
  size_t vsp = rsp;
  size_t vip = 0;
  // Force the relocation address to 0
  REL_ADDR = 0;
  // Execute the virtualized code
  vip = HelperStub(
    rax, rbx, rcx, rdx, rsi, rdi,
    rbp, rsp, r8, r9, r10, r11,
    r12, r13, r14, r15, flags,
    KEY_STUB, RET_ADDR, REL_ADDR,
    vsp, vip, vmregs, slots);
  // Return the next address(es)
  return vip;
}


```

The `VirtualRegister` structure is meant to represent a `VmRegister`, divided in smaller sub-chunks that are going to be accessed by the `VmHandlers` in ways that don’t necessarily match the access to the subregisters on the x64 architecture. As an example, virtualizing the 64 bits `bswap` instruction will yield `VmHandlers` accessing all the word sub-chunks of a `VmRegister`. The `__attribute__((packed))` is meant to generate a structure without padding bytes, matching the exact data layout used by a `VmRegister`.

The `rref` definition is a convenience type adopted in the definition of the arguments used by the helper functions, that, once compiled to LLVM-IR, will generate a pointer parameter with a [noalias](https://llvm.org/docs/LangRef.html#noalias) attribute. The `noalias` attribute is hinting to the compiler that any memory access happening inside the function that is not dereferencing a pointer derived from the pointer parameter, is guaranteed not to alias with a memory access dereferencing a pointer derived from the pointer parameter.

The `RAM`, `GS` and `FS` array definitions are convenience zero-length arrays that we can use to generate indexed memory accesses to a generic memory slot (stack segment, data segment), `GS` segment and `FS` segment. The accesses will be generated as [getelementptr](https://llvm.org/doxygen/classllvm_1_1GetElementPtrInst.html) instructions and LLVM will automatically treat a pointer with base `RAM` as not aliasing with a pointer with base `GS` or `FS`, which is extremely convenient to us.

The `HelperStub` function prototype is a convenience declaration that we’ll be able to use in the lifter to represent a single `VmBlock`. It accepts as parameters the sequence of general purpose register pointers, the flags register pointer, three key values (`KEY_STUB`, `RET_ADDR`, `REL_ADDR`) pushed by each `VmEnter`, the virtual stack pointer, the virtual program counter, the `VmRegisters` pointer and the `VmPassingSlots` pointer.

The `HelperFunction` function definition is a convenience template that we’ll be able to use in the lifter to represent a single `VmStub`. It accepts as parameters the sequence of general purpose register pointers, the flags register pointer and the three key values (`KEY_STUB`, `RET_ADDR`, `REL_ADDR`) pushed by each `VmEnter`. The body is declaring an array of 30 `VmRegisters`, an array of 19 `VmPassingSlots`, the virtual stack pointer and the virtual program counter. Once compiled to LLVM-IR they’ll be turned into [alloca](https://llvm.org/docs/LangRef.html#alloca-instruction) declarations (stack frame allocations), guaranteed not to alias with other pointers used into the function and that will be automatically released at the end of the function scope. As a convenience we are setting the `REL_ADDR` to `0`, but that can be dynamically set to the proper `REL_ADDR` provided by the user according to the needs of the binary under analysis. Last but not least, we are issuing the call to the `HelperStub` function, passing all the needed parameters and obtaining as output the updated instruction pointer, that, in turn, will be returned by the `HelperFunction` too.

The global variable and function declarations are marked as `extern "C"` to avoid any form of name mangling. In fact we want to be able to fetch them from the dynamically loaded LLVM-IR [Module](https://llvm.org/doxygen/classllvm_1_1Module.html) using functions like `getGlobalVariable` and `getFunction`.

The compiled and optimized LLVM-IR code for the described C++ definitions follows:

```
%struct.VirtualRegister = type { %union.anon }
%union.anon = type { i64 }
%struct.anon = type { i8, i8, i8, i8, i8, i8, i8, i8 }

@RAM = external local_unnamed_addr global [0 x i8], align 1
@GS = external local_unnamed_addr global [0 x i8], align 1
@FS = external local_unnamed_addr global [0 x i8], align 1

declare i64 @HelperStub(i64* nonnull align 8 dereferenceable(8), i64* nonnull align 8 dereferenceable(8), i64* nonnull align 8 dereferenceable(8), i64* nonnull align 8 dereferenceable(8), i64* nonnull align 8 dereferenceable(8), i64* nonnull align 8 dereferenceable(8), i64* nonnull align 8 dereferenceable(8), i64* nonnull align 8 dereferenceable(8), i64* nonnull align 8 dereferenceable(8), i64* nonnull align 8 dereferenceable(8), i64* nonnull align 8 dereferenceable(8), i64* nonnull align 8 dereferenceable(8), i64* nonnull align 8 dereferenceable(8), i64* nonnull align 8 dereferenceable(8), i64* nonnull align 8 dereferenceable(8), i64* nonnull align 8 dereferenceable(8), i64* nonnull align 8 dereferenceable(8), i64, i64, i64, i64* nonnull align 8 dereferenceable(8), i64* nonnull align 8 dereferenceable(8), %struct.VirtualRegister*, i64*) local_unnamed_addr

define i64 @HelperFunction(i64* noalias nonnull align 8 dereferenceable(8) %rax, i64* noalias nonnull align 8 dereferenceable(8) %rbx, i64* noalias nonnull align 8 dereferenceable(8) %rcx, i64* noalias nonnull align 8 dereferenceable(8) %rdx, i64* noalias nonnull align 8 dereferenceable(8) %rsi, i64* noalias nonnull align 8 dereferenceable(8) %rdi, i64* noalias nonnull align 8 dereferenceable(8) %rbp, i64* noalias nonnull align 8 dereferenceable(8) %rsp, i64* noalias nonnull align 8 dereferenceable(8) %r8, i64* noalias nonnull align 8 dereferenceable(8) %r9, i64* noalias nonnull align 8 dereferenceable(8) %r10, i64* noalias nonnull align 8 dereferenceable(8) %r11, i64* noalias nonnull align 8 dereferenceable(8) %r12, i64* noalias nonnull align 8 dereferenceable(8) %r13, i64* noalias nonnull align 8 dereferenceable(8) %r14, i64* noalias nonnull align 8 dereferenceable(8) %r15, i64* noalias nonnull align 8 dereferenceable(8) %flags, i64 %KEY_STUB, i64 %RET_ADDR, i64 %REL_ADDR) local_unnamed_addr {
entry:
  %vmregs = alloca [30 x %struct.VirtualRegister], align 16
  %slots = alloca [30 x i64], align 16
  %vip = alloca i64, align 8
  %0 = bitcast [30 x %struct.VirtualRegister]* %vmregs to i8*
  call void @llvm.memset.p0i8.i64(i8* nonnull align 16 dereferenceable(240) %0, i8 0, i64 240, i1 false)
  %1 = bitcast [30 x i64]* %slots to i8*
  call void @llvm.memset.p0i8.i64(i8* nonnull align 16 dereferenceable(240) %1, i8 0, i64 240, i1 false)
  %2 = bitcast i64* %vip to i8*
  store i64 0, i64* %vip, align 8
  %arraydecay = getelementptr inbounds [30 x %struct.VirtualRegister], [30 x %struct.VirtualRegister]* %vmregs, i64 0, i64 0
  %arraydecay1 = getelementptr inbounds [30 x i64], [30 x i64]* %slots, i64 0, i64 0
  %call = call i64 @HelperStub(i64* nonnull align 8 dereferenceable(8) %rax, i64* nonnull align 8 dereferenceable(8) %rbx, i64* nonnull align 8 dereferenceable(8) %rcx, i64* nonnull align 8 dereferenceable(8) %rdx, i64* nonnull align 8 dereferenceable(8) %rsi, i64* nonnull align 8 dereferenceable(8) %rdi, i64* nonnull align 8 dereferenceable(8) %rbp, i64* nonnull align 8 dereferenceable(8) %rsp, i64* nonnull align 8 dereferenceable(8) %r8, i64* nonnull align 8 dereferenceable(8) %r9, i64* nonnull align 8 dereferenceable(8) %r10, i64* nonnull align 8 dereferenceable(8) %r11, i64* nonnull align 8 dereferenceable(8) %r12, i64* nonnull align 8 dereferenceable(8) %r13, i64* nonnull align 8 dereferenceable(8) %r14, i64* nonnull align 8 dereferenceable(8) %r15, i64* nonnull align 8 dereferenceable(8) %flags, i64 %KEY_STUB, i64 %RET_ADDR, i64 0, i64* nonnull align 8 dereferenceable(8) %rsp, i64* nonnull align 8 dereferenceable(8) %vip, %struct.VirtualRegister* nonnull %arraydecay, i64* nonnull %arraydecay1)
  ret i64 %call
}


```

[Semantics of the handlers](#semantics-of-the-handlers)
-------------------------------------------------------

We can now move on to the implementation of the semantics of the handlers used by VMProtect. As mentioned before, implementing them directly at the LLVM-IR level can be a tedious task, so we’ll proceed with the same C++ to LLVM-IR logic adopted in the previous section.

The following selection of handlers should give an idea of the logic adopted to implement the handlers’ semantics.

### [STACK_PUSH](#stack_push)

To access the stack using the push operation, we define a templated helper function that takes the virtual stack pointer and value to push as parameters.

```
template <typename T> __attribute__((always_inline)) void STACK_PUSH(size_t &vsp, T value) {
  // Update the stack pointer
  vsp -= sizeof(T);
  // Store the value
  std::memcpy(&RAM[vsp], &value, sizeof(T));
}


```

We can see that the virtual stack pointer is decremented using the byte size of the template parameter. Then we proceed to use the `std::memcpy` function to execute a [safe type punning](https://gist.github.com/shafik/848ae25ee209f698763cffee272a58f8#how-do-we-type-pun-correctly) store operation accessing the `RAM` array with the virtual stack pointer as index. The C++ implementation is compiled with `-O3` optimizations, so the function will be inlined (as expected from the `always_inline` attribute) and the `std::memcpy` call will be converted to the proper pointer type cast and store instructions.

### [STACK_POP](#stack_pop)

As expected, also the stack pop operation is defined as a templated helper function that takes the virtual stack pointer as parameter and returns the popped value as output.

```
template <typename T> __attribute__((always_inline)) T STACK_POP(size_t &vsp) {
  // Fetch the value
  T value = 0;
  std::memcpy(&value, &RAM[vsp], sizeof(T));
  // Undefine the stack slot
  T undef = UNDEF<T>();
  std::memcpy(&RAM[vsp], &undef, sizeof(T));
  // Update the stack pointer
  vsp += sizeof(T);
  // Return the value
  return value;
}


```

We can see that the value is read from the stack using the same `std::memcpy` logic explained above, an undefined value is written to the current stack slot and the virtual stack pointer is incremented using the byte size of the template parameter. As in the previous case, the `-O3` optimizations will take care of inlining and lowering the `std::memcpy` call.

### [ADD](#add)

Being a stack machine, we know that it is going to pop the two input operands from the top of the stack, add them together, calculate the updated flags and push the result and the flags back to the stack. There are four variations of the addition handler, meant to handle 8/16/32/64 bits operands, with the peculiarity that the 8 bits case is really popping 16 bits per operand off the stack and pushing a 16 bits result back to the stack to be consistent with the x64 push/pop alignment rules.

From what we just described the only thing we need is the virtual stack pointer, to be able to access the stack.

```
// ADD semantic

template <typename T>
__attribute__((always_inline))
__attribute__((const))
bool AF(T lhs, T rhs, T res) {
  return AuxCarryFlag(lhs, rhs, res);
}

template <typename T>
__attribute__((always_inline))
__attribute__((const))
bool PF(T res) {
  return ParityFlag(res);
}

template <typename T>
__attribute__((always_inline))
__attribute__((const))
bool ZF(T res) {
  return ZeroFlag(res);
}

template <typename T>
__attribute__((always_inline))
__attribute__((const))
bool SF(T res) {
  return SignFlag(res);
}

template <typename T>
__attribute__((always_inline))
__attribute__((const))
bool CF_ADD(T lhs, T rhs, T res) {
  return Carry<tag_add>::Flag(lhs, rhs, res);
}

template <typename T>
__attribute__((always_inline))
__attribute__((const))
bool OF_ADD(T lhs, T rhs, T res) {
  return Overflow<tag_add>::Flag(lhs, rhs, res);
}

template <typename T>
__attribute__((always_inline))
void ADD_FLAGS(size_t &flags, T lhs, T rhs, T res) {
  // Calculate the flags
  bool cf = CF_ADD(lhs, rhs, res);
  bool pf = PF(res);
  bool af = AF(lhs, rhs, res);
  bool zf = ZF(res);
  bool sf = SF(res);
  bool of = OF_ADD(lhs, rhs, res);
  // Update the flags
  UPDATE_EFLAGS(flags, cf, pf, af, zf, sf, of);
}

template <typename T>
__attribute__((always_inline))
void ADD(size_t &vsp) {
  // Check if it's 'byte' size
  bool isByte = (sizeof(T) == 1);
  // Initialize the operands
  T op1 = 0;
  T op2 = 0;
  // Fetch the operands
  if (isByte) {
    op1 = Trunc(STACK_POP<uint16_t>(vsp));
    op2 = Trunc(STACK_POP<uint16_t>(vsp));
  } else {
    op1 = STACK_POP<T>(vsp);
    op2 = STACK_POP<T>(vsp);
  }
  // Calculate the add
  T res = UAdd(op1, op2);
  // Calculate the flags
  size_t flags = 0;
  ADD_FLAGS(flags, op1, op2, res);
  // Save the result
  if (isByte) {
    STACK_PUSH<uint16_t>(vsp, ZExt(res));
  } else {
    STACK_PUSH<T>(vsp, res);
  }
  // 7. Save the flags
  STACK_PUSH<size_t>(vsp, flags);
}

DEFINE_SEMANTIC_64(ADD_64) = ADD<uint64_t>;
DEFINE_SEMANTIC(ADD_32) = ADD<uint32_t>;
DEFINE_SEMANTIC(ADD_16) = ADD<uint16_t>;
DEFINE_SEMANTIC(ADD_8) = ADD<uint8_t>;


```

We can see that the function definition is templated with a `T` parameter that is internally used to generate the properly-sized stack accesses executed by the `STACK_PUSH` and `STACK_POP` helpers defined above. Additionally we are taking care of truncating and zero extending the special 8 bits case. Finally, after the unsigned addition took place, we rely on Remill’s semantically proven flag computations to calculate the fresh flags before pushing them to the stack.

The other binary and arithmetic operations are implemented following the same structure, with the correct operands access and flag computations.

### [PUSH_VMREG](#push_vmreg)

This handler is meant to fetch the value stored in a `VmRegister` and push it to the stack. The value can also be a sub-chunk of the virtual register, not necessarily starting from the base of the `VmRegister` slot. Therefore the function arguments are going to be the virtual stack pointer and the value of the `VmRegister`. The template is additionally defining the size of the pushed value and the offset from the `VmRegister` slot base.

```
template <size_t Size, size_t Offset>
__attribute__((always_inline)) void PUSH_VMREG(size_t &vsp, VirtualRegister vmreg) {
  // Update the stack pointer
  vsp -= ((Size != 8) ? (Size / 8) : ((Size / 8) * 2));
  // Select the proper element of the virtual register
  if constexpr (Size == 64) {
    std::memcpy(&RAM[vsp], &vmreg.qword, sizeof(uint64_t));
  } else if constexpr (Size == 32) {
    if constexpr (Offset == 0) {
      std::memcpy(&RAM[vsp], &vmreg.dword.d0, sizeof(uint32_t));
    } else if constexpr (Offset == 1) {
      std::memcpy(&RAM[vsp], &vmreg.dword.d1, sizeof(uint32_t));
    }
  } else if constexpr (Size == 16) {
    if constexpr (Offset == 0) {
      std::memcpy(&RAM[vsp], &vmreg.word.w0, sizeof(uint16_t));
    } else if constexpr (Offset == 1) {
      std::memcpy(&RAM[vsp], &vmreg.word.w1, sizeof(uint16_t));
    } else if constexpr (Offset == 2) {
      std::memcpy(&RAM[vsp], &vmreg.word.w2, sizeof(uint16_t));
    } else if constexpr (Offset == 3) {
      std::memcpy(&RAM[vsp], &vmreg.word.w3, sizeof(uint16_t));
    }
  } else if constexpr (Size == 8) {
    if constexpr (Offset == 0) {
      uint16_t byte = ZExt(vmreg.byte.b0);
      std::memcpy(&RAM[vsp], &byte, sizeof(uint16_t));
    } else if constexpr (Offset == 1) {
      uint16_t byte = ZExt(vmreg.byte.b1);
      std::memcpy(&RAM[vsp], &byte, sizeof(uint16_t));
    }
    // NOTE: there might be other offsets here, but they were not observed
  }
}

DEFINE_SEMANTIC(PUSH_VMREG_8_LOW) = PUSH_VMREG<8, 0>;
DEFINE_SEMANTIC(PUSH_VMREG_8_HIGH) = PUSH_VMREG<8, 1>;
DEFINE_SEMANTIC(PUSH_VMREG_16_LOWLOW) = PUSH_VMREG<16, 0>;
DEFINE_SEMANTIC(PUSH_VMREG_16_LOWHIGH) = PUSH_VMREG<16, 1>;
DEFINE_SEMANTIC_64(PUSH_VMREG_16_HIGHLOW) = PUSH_VMREG<16, 2>;
DEFINE_SEMANTIC_64(PUSH_VMREG_16_HIGHHIGH) = PUSH_VMREG<16, 3>;
DEFINE_SEMANTIC_64(PUSH_VMREG_32_LOW) = PUSH_VMREG<32, 0>;
DEFINE_SEMANTIC_32(POP_VMREG_32) = POP_VMREG<32, 0>;
DEFINE_SEMANTIC_64(PUSH_VMREG_32_HIGH) = PUSH_VMREG<32, 1>;
DEFINE_SEMANTIC_64(PUSH_VMREG_64) = PUSH_VMREG<64, 0>;


```

We can see how the proper `VmRegister` sub-chunk is accessed based on the size and offset template parameters (e.g. `vmreg.word.w1`, `vmreg.qword`) and how once again the `std::memcpy` is used to implement a safe memory write on the indexed `RAM` array. The virtual stack pointer is also decremented as usual.

### [POP_VMREG](#pop_vmreg)

This handler is meant to pop a value from the stack and store it into a `VmRegister`. The value can also be a sub-chunk of the virtual register, not necessarily starting from the base of the `VmRegister` slot. Therefore the function arguments are going to be the virtual stack pointer and a reference to the `VmRegister` to be updated. As before the template is defining the size of the popped value and the offset into the `VmRegister` slot.

```
template <size_t Size, size_t Offset>
__attribute__((always_inline)) void POP_VMREG(size_t &vsp, VirtualRegister &vmreg) {
  // Fetch and store the value on the virtual register
  if constexpr (Size == 64) {
    uint64_t value = 0;
    std::memcpy(&value, &RAM[vsp], sizeof(uint64_t));
    vmreg.qword = value;
  } else if constexpr (Size == 32) {
    if constexpr (Offset == 0) {
      uint32_t value = 0;
      std::memcpy(&value, &RAM[vsp], sizeof(uint32_t));
      vmreg.qword = ((vmreg.qword & 0xFFFFFFFF00000000) | value);
    } else if constexpr (Offset == 1) {
      uint32_t value = 0;
      std::memcpy(&value, &RAM[vsp], sizeof(uint32_t));
      vmreg.qword = ((vmreg.qword & 0x00000000FFFFFFFF) | UShl(ZExt(value), 32));
    }
  } else if constexpr (Size == 16) {
    if constexpr (Offset == 0) {
      uint16_t value = 0;
      std::memcpy(&value, &RAM[vsp], sizeof(uint16_t));
      vmreg.qword = ((vmreg.qword & 0xFFFFFFFFFFFF0000) | value);
    } else if constexpr (Offset == 1) {
      uint16_t value = 0;
      std::memcpy(&value, &RAM[vsp], sizeof(uint16_t));
      vmreg.qword = ((vmreg.qword & 0xFFFFFFFF0000FFFF) | UShl(ZExtTo<uint64_t>(value), 16));
    } else if constexpr (Offset == 2) {
      uint16_t value = 0;
      std::memcpy(&value, &RAM[vsp], sizeof(uint16_t));
      vmreg.qword = ((vmreg.qword & 0xFFFF0000FFFFFFFF) | UShl(ZExtTo<uint64_t>(value), 32));
    } else if constexpr (Offset == 3) {
      uint16_t value = 0;
      std::memcpy(&value, &RAM[vsp], sizeof(uint16_t));
      vmreg.qword = ((vmreg.qword & 0x0000FFFFFFFFFFFF) | UShl(ZExtTo<uint64_t>(value), 48));
    }
  } else if constexpr (Size == 8) {
    if constexpr (Offset == 0) {
      uint16_t byte = 0;
      std::memcpy(&byte, &RAM[vsp], sizeof(uint16_t));
      vmreg.byte.b0 = Trunc(byte);
    } else if constexpr (Offset == 1) {
      uint16_t byte = 0;
      std::memcpy(&byte, &RAM[vsp], sizeof(uint16_t));
      vmreg.byte.b1 = Trunc(byte);
    }
    // NOTE: there might be other offsets here, but they were not observed
  }
  // Clear the value on the stack
  if constexpr (Size == 64) {
    uint64_t undef = UNDEF<uint64_t>();
    std::memcpy(&RAM[vsp], &undef, sizeof(uint64_t));
  } else if constexpr (Size == 32) {
    uint32_t undef = UNDEF<uint32_t>();
    std::memcpy(&RAM[vsp], &undef, sizeof(uint32_t));
  } else if constexpr (Size == 16) {
    uint16_t undef = UNDEF<uint16_t>();
    std::memcpy(&RAM[vsp], &undef, sizeof(uint16_t));
  } else if constexpr (Size == 8) {
    uint16_t undef = UNDEF<uint16_t>();
    std::memcpy(&RAM[vsp], &undef, sizeof(uint16_t));
  }
  // Update the stack pointer
  vsp += ((Size != 8) ? (Size / 8) : ((Size / 8) * 2));
}

DEFINE_SEMANTIC(POP_VMREG_8_LOW) = POP_VMREG<8, 0>;
DEFINE_SEMANTIC(POP_VMREG_8_HIGH) = POP_VMREG<8, 1>;
DEFINE_SEMANTIC(POP_VMREG_16_LOWLOW) = POP_VMREG<16, 0>;
DEFINE_SEMANTIC(POP_VMREG_16_LOWHIGH) = POP_VMREG<16, 1>;
DEFINE_SEMANTIC_64(POP_VMREG_16_HIGHLOW) = POP_VMREG<16, 2>;
DEFINE_SEMANTIC_64(POP_VMREG_16_HIGHHIGH) = POP_VMREG<16, 3>;
DEFINE_SEMANTIC_64(POP_VMREG_32_LOW) = POP_VMREG<32, 0>;
DEFINE_SEMANTIC_64(POP_VMREG_32_HIGH) = POP_VMREG<32, 1>;
DEFINE_SEMANTIC_64(POP_VMREG_64) = POP_VMREG<64, 0>;


```

In this case we can see that the update operation on the sub-chunks of the `VmRegister` is being done with some masking, shifting and zero extensions. This is to help LLVM with merging smaller integer values into a bigger integer value, whenever possible. As we saw in the `STACK_POP` operation, we are writing an undefined value to the current stack slot. Finally we are incrementing the virtual stack pointer.

### [LOAD and LOAD_GS](#load-and-load_gs)

Generically speaking the `LOAD` handler is meant to pop an address from the stack, dereference it to load a value from one of the program segments and push the retrieved value to the top of the stack.

The following C++ snippet shows the implementation of a memory load from a generic memory pointer (e.g. SS or DS segments) and from the GS segment:

```
template <typename T> __attribute__((always_inline)) void LOAD(size_t &vsp) {
  // Check if it's 'byte' size
  bool isByte = (sizeof(T) == 1);
  // Pop the address
  size_t address = STACK_POP<size_t>(vsp);
  // Load the value
  T value = 0;
  std::memcpy(&value, &RAM[address], sizeof(T));
  // Save the result
  if (isByte) {
    STACK_PUSH<uint16_t>(vsp, ZExt(value));
  } else {
    STACK_PUSH<T>(vsp, value);
  }
}

DEFINE_SEMANTIC_64(LOAD_SS_64) = LOAD<uint64_t>;
DEFINE_SEMANTIC(LOAD_SS_32) = LOAD<uint32_t>;
DEFINE_SEMANTIC(LOAD_SS_16) = LOAD<uint16_t>;
DEFINE_SEMANTIC(LOAD_SS_8) = LOAD<uint8_t>;

DEFINE_SEMANTIC_64(LOAD_DS_64) = LOAD<uint64_t>;
DEFINE_SEMANTIC(LOAD_DS_32) = LOAD<uint32_t>;
DEFINE_SEMANTIC(LOAD_DS_16) = LOAD<uint16_t>;
DEFINE_SEMANTIC(LOAD_DS_8) = LOAD<uint8_t>;

template <typename T> __attribute__((always_inline)) void LOAD_GS(size_t &vsp) {
  // Check if it's 'byte' size
  bool isByte = (sizeof(T) == 1);
  // Pop the address
  size_t address = STACK_POP<size_t>(vsp);
  // Load the value
  T value = 0;
  std::memcpy(&value, &GS[address], sizeof(T));
  // Save the result
  if (isByte) {
    STACK_PUSH<uint16_t>(vsp, ZExt(value));
  } else {
    STACK_PUSH<T>(vsp, value);
  }
}

DEFINE_SEMANTIC_64(LOAD_GS_64) = LOAD_GS<uint64_t>;
DEFINE_SEMANTIC(LOAD_GS_32) = LOAD_GS<uint32_t>;
DEFINE_SEMANTIC(LOAD_GS_16) = LOAD_GS<uint16_t>;
DEFINE_SEMANTIC(LOAD_GS_8) = LOAD_GS<uint8_t>;


```

By now the process should be clear. The only difference is the accessed zero-length array that will end up as base of the `getelementptr` instruction, which will directly reflect on the aliasing information that LLVM will be able to infer. The same kind of logic is applied to all the read or write memory accesses to the different segments.

### [DEFINE_SEMANTIC](#define_semantic)

In the code snippets of this section you may have noticed three macros named `DEFINE_SEMANTIC_64`, `DEFINE_SEMANTIC_32` and `DEFINE_SEMANTIC`. They are the umpteenth trick borrowed from Remill and are meant to generate global variables with unmangled names, pointing to the function definition of the specialized template handlers. As an example, the `ADD` semantic definition for the 8/16/32/64 bits cases looks like this at the LLVM-IR level:

```
@SEM_ADD_64 = dso_local constant void (i64*)* @_Z3ADDIyEvRm, align 8
@SEM_ADD_32 = dso_local constant void (i64*)* @_Z3ADDIjEvRm, align 8
@SEM_ADD_16 = dso_local constant void (i64*)* @_Z3ADDItEvRm, align 8
@SEM_ADD_8 = dso_local constant void (i64*)* @_Z3ADDIhEvRm, align 8


```

### [UNDEF](#undef)

In the code snippets of this section you may also have noticed the usage of a function called `UNDEF`. This function is used to store a fictitious `__undef` value after each pop from the stack. This is done to signal to LLVM that the popped value is no longer needed after being popped from the stack.

The `__undef` value is modeled as a global variable, which during the first phase of the optimization pipeline will be used by passes like [DSE](https://llvm.org/doxygen/DeadStoreElimination_8cpp.html) to kill overlapping post-dominated dead stores and it’ll be replaced with a real [undef](https://llvm.org/doxygen/classllvm_1_1UndefValue.html) value near the end of the optimization pipeline such that the related store instruction will be gone on the final optimized LLVM-IR function.

[Lifting a basic block](#lifting-a-basic-block)
-----------------------------------------------

We now have a bunch of templates, structures and helper functions, but how do we actually end up lifting some virtualized code?

The high level idea is the following:

*   A new LLVM-IR function with the `HelperStub` signature is generated;
*   The function’s body is populated with [call](https://llvm.org/doxygen/classllvm_1_1CallInst.html) instructions to the `VmHandler` helper functions fed with the proper arguments (obtained from the `HelperStub` parameters);
*   The optimization pipeline is executed on the function, resulting in the inlining of all the helper functions (that are marked `always_inline`) and in the propagation of the values;
*   The updated state of the `VmRegisters`, `VmPassingSlots` and stores to the segments is optimized, removing most of the obfuscation patterns used by VMProtect;
*   The updated state of the virtual stack pointer and virtual instruction pointer is computed.

A fictitious example of a full pipeline based on the `HelperStub` function, implemented at the C++ level and optimized to obtain propagated LLVM-IR code follows:

```
extern "C" __attribute__((always_inline)) size_t SimpleExample_HelperStub(
  rptr rax, rptr rbx, rptr rcx,
  rptr rdx, rptr rsi, rptr rdi,
  rptr rbp, rptr rsp, rptr r8,
  rptr r9, rptr r10, rptr r11,
  rptr r12, rptr r13, rptr r14,
  rptr r15, rptr flags,
  size_t KEY_STUB, size_t RET_ADDR, size_t REL_ADDR, rptr vsp,
  rptr vip, VirtualRegister *__restrict__ vmregs,
  size_t *__restrict__ slots) {

  PUSH_REG(vsp, rax);
  PUSH_REG(vsp, rbx);
  POP_VMREG<64, 0>(vsp, vmregs[1]);
  POP_VMREG<64, 0>(vsp, vmregs[0]);
  PUSH_VMREG<64, 0>(vsp, vmregs[0]);
  PUSH_VMREG<64, 0>(vsp, vmregs[1]);
  ADD<uint64_t>(vsp);
  POP_VMREG<64, 0>(vsp, vmregs[2]);
  POP_VMREG<64, 0>(vsp, vmregs[3]);
  PUSH_VMREG<64, 0>(vsp, vmregs[3]);
  POP_REG(vsp, rax);

  return vip;
}


```

The C++ `HelperStub` function with calls to the handlers. This only serves as an example, normally the LLVM-IR for this is automatically generated from VM bytecode.

```
define dso_local i64 @SimpleExample_HelperStub(i64* noalias nonnull align 8 dereferenceable(8) %rax, i64* noalias nonnull align 8 dereferenceable(8) %rbx, i64* noalias nonnull align 8 dereferenceable(8) %rcx, i64* noalias nonnull align 8 dereferenceable(8) %rdx, i64* noalias nonnull align 8 dereferenceable(8) %rsi, i64* noalias nonnull align 8 dereferenceable(8) %rdi, i64* noalias nonnull align 8 dereferenceable(8) %rbp, i64* noalias nonnull align 8 dereferenceable(8) %rsp, i64* noalias nonnull align 8 dereferenceable(8) %r8, i64* noalias nonnull align 8 dereferenceable(8) %r9, i64* noalias nonnull align 8 dereferenceable(8) %r10, i64* noalias nonnull align 8 dereferenceable(8) %r11, i64* noalias nonnull align 8 dereferenceable(8) %r12, i64* noalias nonnull align 8 dereferenceable(8) %r13, i64* noalias nonnull align 8 dereferenceable(8) %r14, i64* noalias nonnull align 8 dereferenceable(8) %r15, i64* noalias nonnull align 8 dereferenceable(8) %flags, i64 %KEY_STUB, i64 %RET_ADDR, i64 %REL_ADDR, i64* noalias nonnull align 8 dereferenceable(8) %vsp, i64* noalias nonnull align 8 dereferenceable(8) %vip, %struct.VirtualRegister* noalias %vmregs, i64* noalias %slots) local_unnamed_addr {
entry:
  %rax.addr = alloca i64*, align 8
  %rbx.addr = alloca i64*, align 8
  %rcx.addr = alloca i64*, align 8
  %rdx.addr = alloca i64*, align 8
  %rsi.addr = alloca i64*, align 8
  %rdi.addr = alloca i64*, align 8
  %rbp.addr = alloca i64*, align 8
  %rsp.addr = alloca i64*, align 8
  %r8.addr = alloca i64*, align 8
  %r9.addr = alloca i64*, align 8
  %r10.addr = alloca i64*, align 8
  %r11.addr = alloca i64*, align 8
  %r12.addr = alloca i64*, align 8
  %r13.addr = alloca i64*, align 8
  %r14.addr = alloca i64*, align 8
  %r15.addr = alloca i64*, align 8
  %flags.addr = alloca i64*, align 8
  %KEY_STUB.addr = alloca i64, align 8
  %RET_ADDR.addr = alloca i64, align 8
  %REL_ADDR.addr = alloca i64, align 8
  %vsp.addr = alloca i64*, align 8
  %vip.addr = alloca i64*, align 8
  %vmregs.addr = alloca %struct.VirtualRegister*, align 8
  %slots.addr = alloca i64*, align 8
  %agg.tmp = alloca %struct.VirtualRegister, align 1
  %agg.tmp4 = alloca %struct.VirtualRegister, align 1
  %agg.tmp10 = alloca %struct.VirtualRegister, align 1
  store i64* %rax, i64** %rax.addr, align 8
  store i64* %rbx, i64** %rbx.addr, align 8
  store i64* %rcx, i64** %rcx.addr, align 8
  store i64* %rdx, i64** %rdx.addr, align 8
  store i64* %rsi, i64** %rsi.addr, align 8
  store i64* %rdi, i64** %rdi.addr, align 8
  store i64* %rbp, i64** %rbp.addr, align 8
  store i64* %rsp, i64** %rsp.addr, align 8
  store i64* %r8, i64** %r8.addr, align 8
  store i64* %r9, i64** %r9.addr, align 8
  store i64* %r10, i64** %r10.addr, align 8
  store i64* %r11, i64** %r11.addr, align 8
  store i64* %r12, i64** %r12.addr, align 8
  store i64* %r13, i64** %r13.addr, align 8
  store i64* %r14, i64** %r14.addr, align 8
  store i64* %r15, i64** %r15.addr, align 8
  store i64* %flags, i64** %flags.addr, align 8
  store i64 %KEY_STUB, i64* %KEY_STUB.addr, align 8
  store i64 %RET_ADDR, i64* %RET_ADDR.addr, align 8
  store i64 %REL_ADDR, i64* %REL_ADDR.addr, align 8
  store i64* %vsp, i64** %vsp.addr, align 8
  store i64* %vip, i64** %vip.addr, align 8
  store %struct.VirtualRegister* %vmregs, %struct.VirtualRegister** %vmregs.addr, align 8
  store i64* %slots, i64** %slots.addr, align 8
  %0 = load i64*, i64** %vsp.addr, align 8
  %1 = load i64*, i64** %rax.addr, align 8
  %2 = load i64, i64* %1, align 8
  call void @_Z8PUSH_REGRmm(i64* nonnull align 8 dereferenceable(8) %0, i64 %2)
  %3 = load i64*, i64** %vsp.addr, align 8
  %4 = load i64*, i64** %rbx.addr, align 8
  %5 = load i64, i64* %4, align 8
  call void @_Z8PUSH_REGRmm(i64* nonnull align 8 dereferenceable(8) %3, i64 %5)
  %6 = load i64*, i64** %vsp.addr, align 8
  %7 = load %struct.VirtualRegister*, %struct.VirtualRegister** %vmregs.addr, align 8
  %arrayidx = getelementptr inbounds %struct.VirtualRegister, %struct.VirtualRegister* %7, i64 1
  call void @_Z9POP_VMREGILm64ELm0EEvRmR15VirtualRegister(i64* nonnull align 8 dereferenceable(8) %6, %struct.VirtualRegister* nonnull align 1 dereferenceable(8) %arrayidx)
  %8 = load i64*, i64** %vsp.addr, align 8
  %9 = load %struct.VirtualRegister*, %struct.VirtualRegister** %vmregs.addr, align 8
  %arrayidx1 = getelementptr inbounds %struct.VirtualRegister, %struct.VirtualRegister* %9, i64 0
  call void @_Z9POP_VMREGILm64ELm0EEvRmR15VirtualRegister(i64* nonnull align 8 dereferenceable(8) %8, %struct.VirtualRegister* nonnull align 1 dereferenceable(8) %arrayidx1)
  %10 = load i64*, i64** %vsp.addr, align 8
  %11 = load %struct.VirtualRegister*, %struct.VirtualRegister** %vmregs.addr, align 8
  %arrayidx2 = getelementptr inbounds %struct.VirtualRegister, %struct.VirtualRegister* %11, i64 0
  %12 = bitcast %struct.VirtualRegister* %agg.tmp to i8*
  %13 = bitcast %struct.VirtualRegister* %arrayidx2 to i8*
  call void @llvm.memcpy.p0i8.p0i8.i64(i8* align 1 %12, i8* align 1 %13, i64 8, i1 false)
  %coerce.dive = getelementptr inbounds %struct.VirtualRegister, %struct.VirtualRegister* %agg.tmp, i32 0, i32 0
  %coerce.dive3 = getelementptr inbounds %union.anon, %union.anon* %coerce.dive, i32 0, i32 0
  %14 = load i64, i64* %coerce.dive3, align 1
  call void @_Z10PUSH_VMREGILm64ELm0EEvRm15VirtualRegister(i64* nonnull align 8 dereferenceable(8) %10, i64 %14)
  %15 = load i64*, i64** %vsp.addr, align 8
  %16 = load %struct.VirtualRegister*, %struct.VirtualRegister** %vmregs.addr, align 8
  %arrayidx5 = getelementptr inbounds %struct.VirtualRegister, %struct.VirtualRegister* %16, i64 1
  %17 = bitcast %struct.VirtualRegister* %agg.tmp4 to i8*
  %18 = bitcast %struct.VirtualRegister* %arrayidx5 to i8*
  call void @llvm.memcpy.p0i8.p0i8.i64(i8* align 1 %17, i8* align 1 %18, i64 8, i1 false)
  %coerce.dive6 = getelementptr inbounds %struct.VirtualRegister, %struct.VirtualRegister* %agg.tmp4, i32 0, i32 0
  %coerce.dive7 = getelementptr inbounds %union.anon, %union.anon* %coerce.dive6, i32 0, i32 0
  %19 = load i64, i64* %coerce.dive7, align 1
  call void @_Z10PUSH_VMREGILm64ELm0EEvRm15VirtualRegister(i64* nonnull align 8 dereferenceable(8) %15, i64 %19)
  %20 = load i64*, i64** %vsp.addr, align 8
  call void @_Z3ADDIyEvRm(i64* nonnull align 8 dereferenceable(8) %20)
  %21 = load i64*, i64** %vsp.addr, align 8
  %22 = load %struct.VirtualRegister*, %struct.VirtualRegister** %vmregs.addr, align 8
  %arrayidx8 = getelementptr inbounds %struct.VirtualRegister, %struct.VirtualRegister* %22, i64 2
  call void @_Z9POP_VMREGILm64ELm0EEvRmR15VirtualRegister(i64* nonnull align 8 dereferenceable(8) %21, %struct.VirtualRegister* nonnull align 1 dereferenceable(8) %arrayidx8)
  %23 = load i64*, i64** %vsp.addr, align 8
  %24 = load %struct.VirtualRegister*, %struct.VirtualRegister** %vmregs.addr, align 8
  %arrayidx9 = getelementptr inbounds %struct.VirtualRegister, %struct.VirtualRegister* %24, i64 3
  call void @_Z9POP_VMREGILm64ELm0EEvRmR15VirtualRegister(i64* nonnull align 8 dereferenceable(8) %23, %struct.VirtualRegister* nonnull align 1 dereferenceable(8) %arrayidx9)
  %25 = load i64*, i64** %vsp.addr, align 8
  %26 = load %struct.VirtualRegister*, %struct.VirtualRegister** %vmregs.addr, align 8
  %arrayidx11 = getelementptr inbounds %struct.VirtualRegister, %struct.VirtualRegister* %26, i64 3
  %27 = bitcast %struct.VirtualRegister* %agg.tmp10 to i8*
  %28 = bitcast %struct.VirtualRegister* %arrayidx11 to i8*
  call void @llvm.memcpy.p0i8.p0i8.i64(i8* align 1 %27, i8* align 1 %28, i64 8, i1 false)
  %coerce.dive12 = getelementptr inbounds %struct.VirtualRegister, %struct.VirtualRegister* %agg.tmp10, i32 0, i32 0
  %coerce.dive13 = getelementptr inbounds %union.anon, %union.anon* %coerce.dive12, i32 0, i32 0
  %29 = load i64, i64* %coerce.dive13, align 1
  call void @_Z10PUSH_VMREGILm64ELm0EEvRm15VirtualRegister(i64* nonnull align 8 dereferenceable(8) %25, i64 %29)
  %30 = load i64*, i64** %vsp.addr, align 8
  %31 = load i64*, i64** %rax.addr, align 8
  call void @_Z7POP_REGRmS_(i64* nonnull align 8 dereferenceable(8) %30, i64* nonnull align 8 dereferenceable(8) %31)
  %32 = load i64*, i64** %vip.addr, align 8
  %33 = load i64, i64* %32, align 8
  ret i64 %33
}


```

The LLVM-IR compiled from the previous C++ `HelperStub` function.

```
define dso_local i64 @SimpleExample_HelperStub(i64* noalias nocapture nonnull align 8 dereferenceable(8) %rax, i64* noalias nocapture nonnull readonly align 8 dereferenceable(8) %rbx, i64* noalias nocapture nonnull readnone align 8 dereferenceable(8) %rcx, i64* noalias nocapture nonnull readnone align 8 dereferenceable(8) %rdx, i64* noalias nocapture nonnull readnone align 8 dereferenceable(8) %rsi, i64* noalias nocapture nonnull readnone align 8 dereferenceable(8) %rdi, i64* noalias nocapture nonnull readnone align 8 dereferenceable(8) %rbp, i64* noalias nocapture nonnull readnone align 8 dereferenceable(8) %rsp, i64* noalias nocapture nonnull readnone align 8 dereferenceable(8) %r8, i64* noalias nocapture nonnull readnone align 8 dereferenceable(8) %r9, i64* noalias nocapture nonnull readnone align 8 dereferenceable(8) %r10, i64* noalias nocapture nonnull readnone align 8 dereferenceable(8) %r11, i64* noalias nocapture nonnull readnone align 8 dereferenceable(8) %r12, i64* noalias nocapture nonnull readnone align 8 dereferenceable(8) %r13, i64* noalias nocapture nonnull readnone align 8 dereferenceable(8) %r14, i64* noalias nocapture nonnull readnone align 8 dereferenceable(8) %r15, i64* noalias nocapture nonnull readnone align 8 dereferenceable(8) %flags, i64 %KEY_STUB, i64 %RET_ADDR, i64 %REL_ADDR, i64* noalias nonnull align 8 dereferenceable(8) %vsp, i64* noalias nocapture nonnull readonly align 8 dereferenceable(8) %vip, %struct.VirtualRegister* noalias nocapture %vmregs, i64* noalias nocapture readnone %slots) local_unnamed_addr {
entry:
  %0 = load i64, i64* %rax, align 8
  %1 = load i64, i64* %vsp, align 8
  %sub.i.i = add i64 %1, -8
  %arrayidx.i.i = getelementptr inbounds [0 x i8], [0 x i8]* @RAM, i64 0, i64 %sub.i.i
  %value.addr.0.arrayidx.sroa_cast.i.i = bitcast i8* %arrayidx.i.i to i64*
  %2 = load i64, i64* %rbx, align 8
  %sub.i.i66 = add i64 %1, -16
  %arrayidx.i.i67 = getelementptr inbounds [0 x i8], [0 x i8]* @RAM, i64 0, i64 %sub.i.i66
  %value.addr.0.arrayidx.sroa_cast.i.i68 = bitcast i8* %arrayidx.i.i67 to i64*
  %qword.i62 = getelementptr inbounds %struct.VirtualRegister, %struct.VirtualRegister* %vmregs, i64 1, i32 0, i32 0
  store i64 %2, i64* %qword.i62, align 1
  %3 = load i64, i64* @__undef, align 8
  %qword.i55 = getelementptr inbounds %struct.VirtualRegister, %struct.VirtualRegister* %vmregs, i64 0, i32 0, i32 0
  store i64 %0, i64* %qword.i55, align 1
  %add.i32.i = add i64 %2, %0
  %cmp.i.i.i.i = icmp ult i64 %add.i32.i, %2
  %cmp1.i.i.i.i = icmp ult i64 %add.i32.i, %0
  %4 = or i1 %cmp.i.i.i.i, %cmp1.i.i.i.i
  %conv.i.i.i.i = trunc i64 %add.i32.i to i32
  %conv.i.i.i.i.i = and i32 %conv.i.i.i.i, 255
  %5 = tail call i32 @llvm.ctpop.i32(i32 %conv.i.i.i.i.i)
  %xor.i.i28.i.i = xor i64 %2, %0
  %xor1.i.i.i.i = xor i64 %xor.i.i28.i.i, %add.i32.i
  %and.i.i.i.i = and i64 %xor1.i.i.i.i, 16
  %cmp.i.i27.i.i = icmp eq i64 %add.i32.i, 0
  %shr.i.i.i.i = lshr i64 %2, 63
  %shr1.i.i.i.i = lshr i64 %0, 63
  %shr2.i.i.i.i = lshr i64 %add.i32.i, 63
  %xor.i.i.i.i = xor i64 %shr2.i.i.i.i, %shr.i.i.i.i
  %xor3.i.i.i.i = xor i64 %shr2.i.i.i.i, %shr1.i.i.i.i
  %add.i.i.i.i = add nuw nsw i64 %xor.i.i.i.i, %xor3.i.i.i.i
  %cmp.i.i25.i.i = icmp eq i64 %add.i.i.i.i, 2
  %conv.i.i.i = zext i1 %4 to i64
  %6 = shl nuw nsw i32 %5, 2
  %7 = and i32 %6, 4
  %8 = xor i32 %7, 4
  %9 = zext i32 %8 to i64
  %shl22.i.i.i = select i1 %cmp.i.i27.i.i, i64 64, i64 0
  %10 = lshr i64 %add.i32.i, 56
  %11 = and i64 %10, 128
  %shl34.i.i.i = select i1 %cmp.i.i25.i.i, i64 2048, i64 0
  %or6.i.i.i = or i64 %11, %shl22.i.i.i
  %and13.i.i.i = or i64 %or6.i.i.i, %and.i.i.i.i
  %or17.i.i.i = or i64 %and13.i.i.i, %conv.i.i.i
  %and25.i.i.i = or i64 %or17.i.i.i, %shl34.i.i.i
  %or29.i.i.i = or i64 %and25.i.i.i, %9
  %qword.i36 = getelementptr inbounds %struct.VirtualRegister, %struct.VirtualRegister* %vmregs, i64 2, i32 0, i32 0
  store i64 %or29.i.i.i, i64* %qword.i36, align 1
  store i64 %3, i64* %value.addr.0.arrayidx.sroa_cast.i.i68, align 1
  %qword.i = getelementptr inbounds %struct.VirtualRegister, %struct.VirtualRegister* %vmregs, i64 3, i32 0, i32 0
  store i64 %add.i32.i, i64* %qword.i, align 1
  store i64 %3, i64* %value.addr.0.arrayidx.sroa_cast.i.i, align 1
  store i64 %add.i32.i, i64* %rax, align 8
  %12 = load i64, i64* %vip, align 8
  ret i64 %12
}


```

The LLVM-IR of the `HelperStub` function with inlined and optimized calls to the handlers

The last snippet is representing all the semantic computations related with a `VmBlock`, as described in the high level overview. Although, if the code we lifted is capturing the whole semantics related with a `VmStub`, we can wrap the `HelperStub` function with the `HelperFunction` function, which enforces the liveness properties described in the _Liveness and aliasing information_ section, enabling us to obtain only the computations updating the host execution context:

```
extern "C" size_t SimpleExample_HelperFunction(
    rptr rax, rptr rbx, rptr rcx,
    rptr rdx, rptr rsi, rptr rdi,
    rptr rbp, rptr rsp, rptr r8,
    rptr r9, rptr r10, rptr r11,
    rptr r12, rptr r13, rptr r14,
    rptr r15, rptr flags, size_t KEY_STUB,
    size_t RET_ADDR, size_t REL_ADDR) {
  // Allocate the temporary virtual registers
  VirtualRegister vmregs[30] = {0};
  // Allocate the temporary passing slots
  size_t slots[30] = {0};
  // Initialize the virtual registers
  size_t vsp = rsp;
  size_t vip = 0;
  // Force the relocation address to 0
  REL_ADDR = 0;
  // Execute the virtualized code
  vip = SimpleExample_HelperStub(
    rax, rbx, rcx, rdx, rsi, rdi,
    rbp, rsp, r8, r9, r10, r11,
    r12, r13, r14, r15, flags,
    KEY_STUB, RET_ADDR, REL_ADDR,
    vsp, vip, vmregs, slots);
  // Return the next address(es)
  return vip;
}


```

The C++ `HelperFunction` function with the call to the `HelperStub` function and the relevant stack frame allocations.

```
define dso_local i64 @SimpleExample_HelperFunction(i64* noalias nocapture nonnull align 8 dereferenceable(8) %rax, i64* noalias nocapture nonnull readonly align 8 dereferenceable(8) %rbx, i64* noalias nocapture nonnull readnone align 8 dereferenceable(8) %rcx, i64* noalias nocapture nonnull readnone align 8 dereferenceable(8) %rdx, i64* noalias nocapture nonnull readnone align 8 dereferenceable(8) %rsi, i64* noalias nocapture nonnull readnone align 8 dereferenceable(8) %rdi, i64* noalias nocapture nonnull readnone align 8 dereferenceable(8) %rbp, i64* noalias nocapture nonnull readonly align 8 dereferenceable(8) %rsp, i64* noalias nocapture nonnull readnone align 8 dereferenceable(8) %r8, i64* noalias nocapture nonnull readnone align 8 dereferenceable(8) %r9, i64* noalias nocapture nonnull readnone align 8 dereferenceable(8) %r10, i64* noalias nocapture nonnull readnone align 8 dereferenceable(8) %r11, i64* noalias nocapture nonnull readnone align 8 dereferenceable(8) %r12, i64* noalias nocapture nonnull readnone align 8 dereferenceable(8) %r13, i64* noalias nocapture nonnull readnone align 8 dereferenceable(8) %r14, i64* noalias nocapture nonnull readnone align 8 dereferenceable(8) %r15, i64* noalias nocapture nonnull align 8 dereferenceable(8) %flags, i64 %KEY_STUB, i64 %RET_ADDR, i64 %REL_ADDR) local_unnamed_addr {
entry:
  %0 = load i64, i64* %rax, align 8
  %1 = load i64, i64* %rbx, align 8
  %add.i32.i.i = add i64 %1, %0
  store i64 %add.i32.i.i, i64* %rax, align 8
  ret i64 0
}


```

The LLVM-IR `HelperFunction` function with fully optimized code.

It can be seen that the example is just pushing the values of the registers `rax` and `rbx`, loading them in `vmregs[0]` and `vmregs[1]` respectively, pushing the `VmRegisters` on the stack, adding them together, popping the updated flags in `vmregs[2]`, popping the addition’s result to `vmregs[3]` and finally pushing `vmregs[3]` on the stack to be popped in the `rax` register at the end. The liveness of the values of the `VmRegisters` ends with the end of the function, hence the updated flags saved in `vmregs[2]` won’t be reflected on the host execution context. Looking at the final snippet we can see that the semantics of the code have been successfully obtained.

In [Part 2](https://secret.club/2021/09/08/vmprotect-llvm-lifting-2.html) we’ll put the described structures and helpers to good use, digging into the details of the virtualized CFG exploration and introducing the basics of the LLVM optimization pipeline.