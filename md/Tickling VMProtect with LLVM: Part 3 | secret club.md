> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [secret.club](https://secret.club/2021/09/08/vmprotect-llvm-lifting-3.html)

> This post will introduce 7 custom passes that, once added to the optimization pipeline, will make the......

This post will introduce 7 custom passes that, once added to the optimization pipeline, will make the overall LLVM-IR output more readable. Some words will be spent on the _unsupported instructions lifting_ and _recompilation_ topics. Finally, the output of 6 devirtualized functions will be shown.

This section will give an overview of some custom passes meant to:

*   Solve VMProtect specific optimization problems;
*   Solve some limitations of existing LLVM passes, but that won’t meet the same quality standard of an official LLVM pass.

[SegmentsAA](#segmentsaa)
-------------------------

This pass falls under the category of the VMProtect specific optimization problems and is probably the most delicate of the section, as it may be feeding LLVM with unsafe assumptions. The aliasing information described in the _Liveness and aliasing information_ section will finally come in handy. In fact, the goal of the pass is to identify the type of two pointers and determine if they can be deemed as not aliasing with one another.

With the structures defined in the previous sections, LLVM is already able to infer that two pointers derived from the following sources don’t alias with one another:

*   general purpose registers
*   `VmRegisters`
*   `VmPassingSlots`
*   `GS` zero-sized array
*   `FS` zero-sized array
*   `RAM` zero-sized array (with constant index)
*   `RAM` zero-sized array (with symbolic index)

Additionally LLVM can also discern between pointers with `RAM` base using a simple symbolic index. For example an access to `[rsp - 0x10]` (local stack slot) will be considered as `NoAlias` when compared with an access to `[rsp + 0x10]` (incoming stack argument).

But LLVM’s alias analysis passes fall short when handling pointers using as base the `RAM` array and employing a more convoluted symbolic index, and the reason for the shortcoming is entirely related to the lack of type and context information that got lost during the compilation to binary.

The pass is inspired by existing implementations ([1](https://blog.tartanllama.xyz/llvm-alias-analysis/), [2](https://llvm.org/doxygen/AMDGPUAliasAnalysis_8h.html), [3](https://github.com/zneak/fcd/blob/master/fcd/pass_regaa.cpp)) that are basing their checks on the identification of pointers belonging to different segments and address spaces.

Slicing the symbolic index used in a `RAM` array access we can discern with high confidence between the following additional `NoAlias` memory accesses:

*   **indirect access**: if the access is a stack argument (`[rsp]` or `[rsp + positive_constant_offset + symbolic_offset]`), a dereferenced general purpose register (`[rax]`) or a nested dereference (`val1 = [rax]`, `val2 = [val1]`); identified as `TyIND` in the code;
*   **local stack slot**: if the access is of the form `[rsp - positive_constant_offset + symbolic_offset]`; identified as `TySS` in the code;
*   **local stack array**: if the access if of the form `[rsp - positive_constant_offset + phi_index]`; identified as `TyARR` in the code.

If the pointer type cannot be reliably detected, an unknown type (identified as `TyUNK` in the code) is being used, and the comparison between the pointers is automatically skipped. If the pass cannot return a `NoAlias` result, the query is passed back to the default alias analysis pipeline.

One could argue that the pass is not really needed, as it is unlikely that the propagation of the sensitive information we need to successfully explore the virtualized CFG is hindered by aliasing issues. In fact, the computation of a conditional branch at the end of a `VmBlock` is guaranteed not to be hindered by a symbolic memory store happening before the jump `VmHandler` accesses the branch destination. But there are some cases where VMProtect pushes the address of the next `VmStub` in one of the first `VmBlocks`, doing memory stores in between and accessing the pushed value only in one or more `VmExits`. That could be a case where discerning between a local stack slot and an indirect access enables the propagation of the pushed address.

Irregardless of the aforementioned issue, that can be solved with some ad-hoc `store-to-load` detection logic, playing around with the alias analysis information that can be fed to LLVM could make the devirtualized code more readable. We have to keep in mind that there may be edge cases where the original code is breaking our assumptions, so having at least a vague idea of the involved pointers accessed at runtime could give us more confidence or force us to err on the safe side, relying solely on the built-in LLVM alias analysis passes.

The assembly snippet shown below has been devirtualized with and without adding the `SegmentsAA` pass to the pipeline. If we are sure that at runtime, before the `push rax` instruction, `rcx` doesn’t contain the value `rsp - 8` (extremely unexpected on benign code), we can safely enable the `SegmentsAA` pass and obtain a cleaner output.

```
start:
  push rax
  mov qword ptr [rsp], 1
  mov qword ptr [rcx], 2
  pop rax


```

The assembly code writing to two possibly aliasing memory slots

```
define dso_local i64 @F_0x14000101f(i64* noalias nonnull align 8 dereferenceable(8) %rax, i64* noalias nonnull align 8 dereferenceable(8) %rbx, i64* noalias nonnull align 8 dereferenceable(8) %rcx, i64* noalias nonnull align 8 dereferenceable(8) %rdx, i64* noalias nonnull align 8 dereferenceable(8) %rsi, i64* noalias nonnull align 8 dereferenceable(8) %rdi, i64* noalias nonnull align 8 dereferenceable(8) %rbp, i64* noalias nonnull align 8 dereferenceable(8) %rsp, i64* noalias nonnull align 8 dereferenceable(8) %r8, i64* noalias nonnull align 8 dereferenceable(8) %r9, i64* noalias nonnull align 8 dereferenceable(8) %r10, i64* noalias nonnull align 8 dereferenceable(8) %r11, i64* noalias nonnull align 8 dereferenceable(8) %r12, i64* noalias nonnull align 8 dereferenceable(8) %r13, i64* noalias nonnull align 8 dereferenceable(8) %r14, i64* noalias nonnull align 8 dereferenceable(8) %r15, i64* noalias nonnull align 8 dereferenceable(8) %flags, i64 %KEY_STUB, i64 %RET_ADDR, i64 %REL_ADDR) #2 {
  %1 = load i64, i64* %rcx, align 8, !alias.scope !22, !noalias !27
  %2 = inttoptr i64 %1 to i64*
  store i64 2, i64* %2, align 1, !noalias !65
  store i64 1, i64* %rax, align 8, !tbaa !4, !alias.scope !66, !noalias !67
  ret i64 5368713278
}


```

The devirtualized code with the `SegmentsAA` pass added to the pipeline, with the assumption that `rcx` differs from `rsp - 8`

```
define dso_local i64 @F_0x14000101f(i64* noalias nonnull align 8 dereferenceable(8) %rax, i64* noalias nonnull align 8 dereferenceable(8) %rbx, i64* noalias nonnull align 8 dereferenceable(8) %rcx, i64* noalias nonnull align 8 dereferenceable(8) %rdx, i64* noalias nonnull align 8 dereferenceable(8) %rsi, i64* noalias nonnull align 8 dereferenceable(8) %rdi, i64* noalias nonnull align 8 dereferenceable(8) %rbp, i64* noalias nonnull align 8 dereferenceable(8) %rsp, i64* noalias nonnull align 8 dereferenceable(8) %r8, i64* noalias nonnull align 8 dereferenceable(8) %r9, i64* noalias nonnull align 8 dereferenceable(8) %r10, i64* noalias nonnull align 8 dereferenceable(8) %r11, i64* noalias nonnull align 8 dereferenceable(8) %r12, i64* noalias nonnull align 8 dereferenceable(8) %r13, i64* noalias nonnull align 8 dereferenceable(8) %r14, i64* noalias nonnull align 8 dereferenceable(8) %r15, i64* noalias nonnull align 8 dereferenceable(8) %flags, i64 %KEY_STUB, i64 %RET_ADDR, i64 %REL_ADDR) #2 {
  %1 = load i64, i64* %rsp, align 8, !tbaa !4, !alias.scope !22, !noalias !27
  %2 = add i64 %1, -8
  %3 = load i64, i64* %rcx, align 8, !alias.scope !65, !noalias !66
  %4 = inttoptr i64 %2 to i64*
  store i64 1, i64* %4, align 1, !noalias !67
  %5 = inttoptr i64 %3 to i64*
  store i64 2, i64* %5, align 1, !noalias !67
  %6 = load i64, i64* %4, align 1, !noalias !67
  store i64 %6, i64* %rax, align 8, !tbaa !4, !alias.scope !68, !noalias !69
  ret i64 5368713278
}


```

The devirtualized code without the `SegmentsAA` pass added to the pipeline and therefore no assumptions fed to LLVM

Alias analysis is a complex topic, and experience thought me that most of the propagation issues happening while using LLVM to deobfuscate some code are related to the LLVM’s alias analysis passes being hinder by some pointer computation. Therefore, having the capability to feed LLVM with context-aware information could be the only way to handle certain types of obfuscation. Beware that other tools you are used to are most likely doing similar “safe” assumptions under the hood (e.g. concolic execution tools using the concrete pointer to answer the aliasing queries).

The takeaway from this section is that, if needed, you can define your own alias analysis callback pass to be integrated in the optimization pipeline in such a way that pre-existing passes can make use of the refined aliasing query results. This is similar to updating IDA’s stack variables with proper type definitions to improve the propagation results.

[KnownIndexSelect](#knownindexselect)
-------------------------------------

This pass falls under the category of the VMProtect specific optimization problems. In fact, whoever looked into VMProtect 3.0.9 knows that the following trick, reimplemented as high level C code for simplicity, is being internally used to select between two branches of a conditional jump.

```
uint64_t ConditionalBranchLogic(uint64_t RFLAGS) {
  // Extracting the ZF flag bit
  uint64_t ConditionBit = (RFLAGS & 0x40) >> 6;
  // Writing the jump destinations
  uint64_t Stack[2] = { 0 };
  Stack[0] = 5369966919;
  Stack[1] = 5369966790;
  // Selecting the correct jump destination
  return Stack[ConditionBit];
}


```

What is really happening at the low level is that the branch destinations are written to adjacent stack slots and then a conditional load, controlled by the previously computed flags, is going to select between one slot or the other to fetch the right jump destination.

LLVM doesn’t automatically see through the conditional load, but it is providing us with all the needed information to write such an optimization ourselves. In fact, the [ValueTracking](https://llvm.org/doxygen/ValueTracking_8cpp.html) analysis exposes the `computeKnownBits` function that we can use to determine if the index used in a `getelementptr` instruction is bound to have just two values.

At this point we can generate two separated [load](https://llvm.org/doxygen/classllvm_1_1LoadInst.html) instructions accessing the stack slots with the inferred indices and feed them to a [select](https://llvm.org/doxygen/classllvm_1_1SelectInst.html) instruction controlled by the index itself. At the next `store-to-load` propagation, LLVM will happily identify the matching `store` and `load` instructions, propagating the constants representing the conditional branch destinations and generating a nice `select` instruction with second and third constant operands.

```
; Matched pattern
%i0 = add i64 %base, %index
%i1 = getelementptr inbounds [0 x i8], [0 x i8]* @RAM, i64 0, i64 %i0
%i2 = bitcast i8* %i1 to i64*
%i3 = load i64, i64* %i2, align 1

; Exploded form
%51 = add i64 %base, 0
%52 = add i64 %base, 8
%53 = getelementptr inbounds [0 x i8], [0 x i8]* @RAM, i64 0, i64 %51
%54 = getelementptr inbounds [0 x i8], [0 x i8]* @RAM, i64 0, i64 %52
%55 = bitcast i8* %53 to i64*
%56 = bitcast i8* %54 to i64*
%57 = load i64, i64* %55, align 8
%58 = load i64, i64* %56, align 8
%59 = icmp eq i64 %index, 0
%60 = select i1 %59, i64 %57, i64 %58

; Simplified select instruction
%59 = icmp eq i64 %index, 0
%60 = select i1 %59, i64 5369966919, i64 5369966790


```

The snippet above shows the matched pattern, its exploded form suitable for the LLVM propagation and its final optimized shape. In this case the `ValueTracking` analysis provided the values `0` and `8` as the only feasible ones for the `%index` value.

A brief discussion about this pass can be found in this [chain of messages](http://lists.llvm.org/pipermail/llvm-dev/2020-June/142426.html) in the LLVM mailing list.

[SynthesizeFlags](#synthesizeflags)
-----------------------------------

This pass falls in between the categories of the VMProtect specific optimization problems and LLVM optimization limitations. In fact, this pass is based on the [enumerative synthesis](https://github.com/google/souper/blob/main/lib/Infer/EnumerativeSynthesis.cpp) logic implemented by Souper, with some minor tweaks to make it more performant for our use-case.

This pass exists because I’m lazy and the fewer ad-hoc patterns I write, the happier I am. The patterns we are talking about are the ones generated by the flag manipulations that VMProtect does when computing the condition for a conditional branch. LLVM already does a good job with simplifying part of the patterns, but to obtain mint-like results we absolutely need to help it a bit.

There’s not much to say about this pass, it is basically invoking Souper’s enumerative synthesis with a selected set of components (`Inst::CtPop`, `Inst::Eq`, `Inst::Ne`, `Inst::Ult`, `Inst::Slt`, `Inst::Ule`, `Inst::Sle`, `Inst::SAddO`, `Inst::UAddO`, `Inst::SSubO`, `Inst::USubO`, `Inst::SMulO`, `Inst::UMulO`), requiring the synthesis of a single instruction, enabling the data-flow pruning option and bounding the LHS candidates to a maximum of 50. Additionally the pass is executing the synthesis only on the `i1` conditions used by the `select` and [br](https://llvm.org/doxygen/classllvm_1_1BranchInst.html) instructions.

This [Godbolt page](https://godbolt.org/z/aYdc3xhn9) shows the devirtualized LLVM-IR output obtained appending the `SynthesizeFlags` pass to the pipeline and the resulting assembly with the properly recompiled conditional jumps. The original assembly code can be seen below. It’s a dummy sequence of instructions where the key piece is the comparison between the `rax` and `rbx` registers that drives the conditional branch `jcc`.

```
start:
  cmp rax, rbx
  jcc label
  and rcx, rdx
  mov qword ptr ds:[rcx], 1
  jmp exit
label:
  xor rcx, rdx
  add qword ptr ds:[rcx], 2
exit:


```

[MemoryCoalescing](#memorycoalescing)
-------------------------------------

This pass falls under the category of the generic LLVM optimization passes that couldn’t possibly be included in the mainline framework because they wouldn’t match the quality criteria of a stable pass. Although the transformations done by this pass are applicable to generic LLVM-IR code, even if the handled cases are most likely to be found in obfuscated code.

Passes like `DSE` already attempt to handle the case where a `store` instruction is partially or completely overlapping with other `store` instructions. Although the more convoluted case of multiple `stores` contributing to the value of a single memory slot are somehow only partially handled.

This pass is focusing on the handling of the case illustrated in the following snippet, where multiple smaller `stores` contribute to the creation of a bigger value subsequently accessed by a single `load` instruction.

```
define dso_local i64 @WhatDidYouDoToMySonYouEvilMonster(i64* noalias nonnull align 8 dereferenceable(8) %rax, i64* noalias nonnull align 8 dereferenceable(8) %rbx, i64* noalias nonnull align 8 dereferenceable(8) %rcx, i64* noalias nonnull align 8 dereferenceable(8) %rdx, i64* noalias nonnull align 8 dereferenceable(8) %rsi, i64* noalias nonnull align 8 dereferenceable(8) %rdi, i64* noalias nonnull align 8 dereferenceable(8) %rbp, i64* noalias nonnull align 8 dereferenceable(8) %rsp, i64* noalias nonnull align 8 dereferenceable(8) %r8, i64* noalias nonnull align 8 dereferenceable(8) %r9, i64* noalias nonnull align 8 dereferenceable(8) %r10, i64* noalias nonnull align 8 dereferenceable(8) %r11, i64* noalias nonnull align 8 dereferenceable(8) %r12, i64* noalias nonnull align 8 dereferenceable(8) %r13, i64* noalias nonnull align 8 dereferenceable(8) %r14, i64* noalias nonnull align 8 dereferenceable(8) %r15, i64* noalias nonnull align 8 dereferenceable(8) %flags, i64 %KEY_STUB, i64 %RET_ADDR, i64 %REL_ADDR) {
  %1 = load i64, i64* %rsp, align 8
  %2 = add i64 %1, -8
  %3 = getelementptr inbounds [0 x i8], [0 x i8]* @RAM, i64 0, i64 %2
  %4 = add i64 %1, -16
  %5 = getelementptr inbounds [0 x i8], [0 x i8]* @RAM, i64 0, i64 %4
  %6 = load i64, i64* %rax, align 8
  %7 = add i64 %1, -10
  %8 = getelementptr inbounds [0 x i8], [0 x i8]* @RAM, i64 0, i64 %7
  %9 = bitcast i8* %8 to i64*
  store i64 %6, i64* %9, align 1
  %10 = bitcast i8* %8 to i32*
  %11 = trunc i64 %6 to i32
  %12 = add i64 %1, -6
  %13 = getelementptr inbounds [0 x i8], [0 x i8]* @RAM, i64 0, i64 %12
  %14 = bitcast i8* %13 to i32*
  store i32 %11, i32* %14, align 1
  %15 = bitcast i8* %13 to i16*
  %16 = trunc i64 %6 to i16
  %17 = add i64 %1, -4
  %18 = add i64 %1, -12
  %19 = getelementptr inbounds [0 x i8], [0 x i8]* @RAM, i64 0, i64 %18
  %20 = bitcast i8* %19 to i64*
  %21 = getelementptr inbounds [0 x i8], [0 x i8]* @RAM, i64 0, i64 %17
  %22 = bitcast i8* %21 to i16*
  %23 = load i16, i16* %22, align 1
  store i16 %23, i16* %15, align 1
  %24 = load i32, i32* %14, align 1
  %25 = shl i32 %24, 8
  %26 = bitcast i8* %21 to i32*
  store i32 %25, i32* %26, align 1
  %27 = bitcast i8* %3 to i16*
  store i16 %16, i16* %27, align 1
  %28 = bitcast i8* %8 to i16*
  store i16 %16, i16* %28, align 1
  %29 = load i32, i32* %10, align 1
  %30 = shl i32 %29, 8
  %31 = bitcast i8* %3 to i32*
  store i32 %30, i32* %31, align 1
  %32 = add i64 %1, -14
  %33 = getelementptr inbounds [0 x i8], [0 x i8]* @RAM, i64 0, i64 %32
  %34 = add i64 %1, -2
  %35 = getelementptr inbounds [0 x i8], [0 x i8]* @RAM, i64 0, i64 %34
  %36 = bitcast i8* %35 to i16*
  %37 = load i16, i16* %36, align 1
  store i16 %37, i16* %27, align 1
  %38 = add i64 %1, -18
  %39 = getelementptr inbounds [0 x i8], [0 x i8]* @RAM, i64 0, i64 %38
  %40 = bitcast i8* %39 to i64*
  store i64 %6, i64* %40, align 1
  %41 = bitcast i8* %39 to i32*
  %42 = bitcast i8* %33 to i16*
  %43 = load i16, i16* %42, align 1
  %44 = bitcast i8* %19 to i16*
  %45 = load i16, i16* %44, align 1
  store i16 %45, i16* %42, align 1
  %46 = bitcast i8* %33 to i32*
  %47 = load i32, i32* %46, align 1
  %48 = shl i32 %47, 8
  %49 = bitcast i8* %19 to i32*
  store i32 %48, i32* %49, align 1
  %50 = bitcast i8* %5 to i16*
  store i16 %43, i16* %50, align 1
  %51 = bitcast i8* %39 to i16*
  store i16 %43, i16* %51, align 1
  %52 = load i32, i32* %41, align 1
  %53 = shl i32 %52, 8
  %54 = bitcast i8* %5 to i32*
  store i32 %53, i32* %54, align 1
  %55 = load i16, i16* %28, align 1
  store i16 %55, i16* %50, align 1
  %56 = load i32, i32* %54, align 1
  store i32 %56, i32* %49, align 1
  %57 = load i64, i64* %20, align 1
  store i64 %57, i64* %rax, align 8
  ret i64 5368713262
}


```

Now, you can arm yourself with patience and manually match all the `store` and `load` operations, or you can trust me when I tell you that all of them are concurring to the creation of a single `i64` value that will be finally saved in the `rax` register.

The pass is working at the intra-block level and it’s relying on the analysis results provided by the `MemorySSA`, [ScalarEvolution](https://llvm.org/doxygen/classllvm_1_1ScalarEvolution.html) and [AAResults](https://llvm.org/doxygen/classllvm_1_1AAResults.html) interfaces to backward walk the definition chains concurring to the creation of the value fetched by each `load` instruction in the block. Doing that, it is filling a structure which keeps track of the aliasing `store` instructions, the stored values, and the offsets and sizes overlapping with the memory slot fetched by each `load`. If a sequence of `store` assignments completely defining the value of the whole memory slot is found, the chain is processed to remove the `store-to-load` indirection. Subsequent passes may then rely on this new indirection-free chain to apply more transformations. As an example the previous LLVM-IR snippet turns in the following optimized LLVM-IR snippet when the `MemoryCoalescing` pass is applied before executing the `InstCombine` pass. Nice huh?

```
define dso_local i64 @ThanksToLLVMMySonBswap64IsBack(i64* noalias nonnull align 8 dereferenceable(8) %rax, i64* noalias nonnull align 8 dereferenceable(8) %rbx, i64* noalias nonnull align 8 dereferenceable(8) %rcx, i64* noalias nonnull align 8 dereferenceable(8) %rdx, i64* noalias nonnull align 8 dereferenceable(8) %rsi, i64* noalias nonnull align 8 dereferenceable(8) %rdi, i64* noalias nonnull align 8 dereferenceable(8) %rbp, i64* noalias nonnull align 8 dereferenceable(8) %rsp, i64* noalias nonnull align 8 dereferenceable(8) %r8, i64* noalias nonnull align 8 dereferenceable(8) %r9, i64* noalias nonnull align 8 dereferenceable(8) %r10, i64* noalias nonnull align 8 dereferenceable(8) %r11, i64* noalias nonnull align 8 dereferenceable(8) %r12, i64* noalias nonnull align 8 dereferenceable(8) %r13, i64* noalias nonnull align 8 dereferenceable(8) %r14, i64* noalias nonnull align 8 dereferenceable(8) %r15, i64* noalias nonnull align 8 dereferenceable(8) %flags, i64 %KEY_STUB, i64 %RET_ADDR, i64 %REL_ADDR) {
  %1 = load i64, i64* %rax, align 8
  %2 = call i64 @llvm.bswap.i64(i64 %1)
  store i64 %2, i64* %rax, align 8
  ret i64 5368713262
}


```

[PartialOverlapDSE](#partialoverlapdse)
---------------------------------------

This pass also falls under the category of the generic LLVM optimization passes that couldn’t possibly be included in the mainline framework because they wouldn’t match the quality criteria of a stable pass. Although the transformations done by this pass are applicable to generic LLVM-IR code, even if the handled cases are most likely to be found in obfuscated code.

Conceptually similar to the `MemoryCoalescing` pass, the goal of this pass is to sweep a function to identify chains of `store` instructions that post-dominate a single `store` instruction and kill its value before it is actually being fetched. Passes like `DSE` are doing a similar job, although limited to some forms of full overlapping caused by multiple `stores` on a single post-dominated `store`.

Applying the `-O3` pipeline to the following example won’t remove the first 64 bits dead `store` at `RAM[%0]`, even if the subsequent 64 bits `stores` at `RAM[%0 - 4]` and `RAM[%0 + 4]` fully overlap it, redefining its value.

```
define dso_local void @DSE(i64 %0) local_unnamed_addr #0 {
  %2 = getelementptr inbounds [0 x i8], [0 x i8]* @RAM, i64 0, i64 %0
  %3 = bitcast i8* %2 to i64*
  store i64 1234605616436508552, i64* %3, align 1
  %4 = add i64 %0, -4
  %5 = getelementptr inbounds [0 x i8], [0 x i8]* @RAM, i64 0, i64 %4
  %6 = bitcast i8* %5 to i64*
  store i64 -6148858396813837381, i64* %6, align 1
  %7 = add i64 %0, 4
  %8 = getelementptr inbounds [0 x i8], [0 x i8]* @RAM, i64 0, i64 %7
  %9 = bitcast i8* %8 to i64*
  store i64 -3689348814455574802, i64* %9, align 1
  ret void
}


```

Adding the `PartialOverlapDSE` pass to the pipeline will identify and kill the first `store`, enabling other passes to eventually kill the chain of computations contributing to the stored value. The built-in `DSE` pass is most likely not executing such a kill because collecting information about multiple overlapping `stores` is an expensive operation.

[PointersHoisting](#pointershoisting)
-------------------------------------

This pass is strictly related to the `IsGuaranteedLoopInvariant` patch I submitted, in fact it is just identifying all the pointers that could be safely hoisted to the entry block because depending solely on values coming directly from the entry block. Applying this kind of transformation prior to the execution of the `DSE` pass may lead to better optimization results.

As an example, consider this devirtualized function containing a rather useless switch case. I’m saying rather useless because each `store` in the case blocks is post-dominated and killed by the `store i32 22, i32* %85` instruction, but LLVM is not going to kill those stores until we move the pointer computation to the entry block.

```
define dso_local i64 @F_0x1400045b0(i64* noalias nonnull align 8 dereferenceable(8) %rax, i64* noalias nonnull align 8 dereferenceable(8) %rbx, i64* noalias nonnull align 8 dereferenceable(8) %rcx, i64* noalias nonnull align 8 dereferenceable(8) %rdx, i64* noalias nonnull align 8 dereferenceable(8) %rsi, i64* noalias nonnull align 8 dereferenceable(8) %rdi, i64* noalias nonnull align 8 dereferenceable(8) %rbp, i64* noalias nonnull align 8 dereferenceable(8) %rsp, i64* noalias nonnull align 8 dereferenceable(8) %r8, i64* noalias nonnull align 8 dereferenceable(8) %r9, i64* noalias nonnull align 8 dereferenceable(8) %r10, i64* noalias nonnull align 8 dereferenceable(8) %r11, i64* noalias nonnull align 8 dereferenceable(8) %r12, i64* noalias nonnull align 8 dereferenceable(8) %r13, i64* noalias nonnull align 8 dereferenceable(8) %r14, i64* noalias nonnull align 8 dereferenceable(8) %r15, i64* noalias nonnull align 8 dereferenceable(8) %flags, i64 %KEY_STUB, i64 %RET_ADDR, i64 %REL_ADDR) {
  %1 = load i64, i64* %rsp, align 8
  %2 = load i64, i64* %rcx, align 8
  %3 = call { i32, i32, i32, i32 } asm "  xchgq  %rbx,${1:q}\0A  cpuid\0A  xchgq  %rbx,${1:q}", "={ax},=r,={cx},={dx},0,~{dirflag},~{fpsr},~{flags}"(i32 1)
  %4 = extractvalue { i32, i32, i32, i32 } %3, 0
  %5 = extractvalue { i32, i32, i32, i32 } %3, 1
  %6 = and i32 %4, 4080
  %7 = icmp eq i32 %6, 4064
  %8 = xor i32 %4, 32
  %9 = select i1 %7, i32 %8, i32 %4
  %10 = and i32 %5, 16777215
  %11 = add i32 %9, %10
  %12 = load i64, i64* bitcast (i8* getelementptr inbounds ([0 x i8], [0 x i8]* @GS, i64 0, i64 96) to i64*), align 1
  %13 = add i64 %12, 288
  %14 = getelementptr inbounds [0 x i8], [0 x i8]* @RAM, i64 0, i64 %13
  %15 = bitcast i8* %14 to i16*
  %16 = load i16, i16* %15, align 1
  %17 = zext i16 %16 to i32
  %18 = load i64, i64* bitcast (i8* getelementptr inbounds ([0 x i8], [0 x i8]* @RAM, i64 0, i64 5369231568) to i64*), align 1
  %19 = shl nuw nsw i32 %17, 7
  %20 = add i64 %18, 32
  %21 = getelementptr inbounds [0 x i8], [0 x i8]* @RAM, i64 0, i64 %20
  %22 = bitcast i8* %21 to i32*
  %23 = load i32, i32* %22, align 1
  %24 = xor i32 %23, 2480
  %25 = add i32 %11, %19
  %26 = trunc i64 %18 to i32
  %27 = add i64 %18, 88
  %28 = xor i32 %25, %26
  br label %29

29:                                               ; preds = %37, %0
  %30 = phi i64 [ %27, %0 ], [ %38, %37 ]
  %31 = phi i32 [ %24, %0 ], [ %39, %37 ]
  %32 = getelementptr inbounds [0 x i8], [0 x i8]* @RAM, i64 0, i64 %30
  %33 = bitcast i8* %32 to i32*
  %34 = load i32, i32* %33, align 1
  %35 = xor i32 %34, 30826
  %36 = icmp eq i32 %28, %35
  br i1 %36, label %41, label %37

37:                                               ; preds = %29
  %38 = add i64 %30, 8
  %39 = add i32 %31, -1
  %40 = icmp eq i32 %31, 1
  br i1 %40, label %86, label %29

41:                                               ; preds = %29
  %42 = add i64 %1, 40
  %43 = getelementptr inbounds [0 x i8], [0 x i8]* @RAM, i64 0, i64 %42
  %44 = bitcast i8* %43 to i32*
  %45 = load i32, i32* %44, align 1
  %46 = add i64 %1, 36
  %47 = getelementptr inbounds [0 x i8], [0 x i8]* @RAM, i64 0, i64 %46
  %48 = bitcast i8* %47 to i32*
  store i32 %45, i32* %48, align 1
  %49 = icmp ugt i32 %45, 5
  %50 = zext i32 %45 to i64
  %51 = add i64 %1, 32
  br i1 %49, label %81, label %52

52:                                               ; preds = %41
  %53 = shl nuw nsw i64 %50, 1
  %54 = and i64 %53, 4294967296
  %55 = sub nsw i64 %50, %54
  %56 = shl nsw i64 %55, 2
  %57 = add nsw i64 %56, 5368964976
  %58 = getelementptr inbounds [0 x i8], [0 x i8]* @RAM, i64 0, i64 %57
  %59 = bitcast i8* %58 to i32*
  %60 = load i32, i32* %59, align 1
  %61 = zext i32 %60 to i64
  %62 = add nuw nsw i64 %61, 5368709120
  switch i32 %60, label %66 [
    i32 1465288, label %63
    i32 1510355, label %78
    i32 1706770, label %75
    i32 2442748, label %72
    i32 2740242, label %69
  ]

63:                                               ; preds = %52
  %64 = getelementptr inbounds [0 x i8], [0 x i8]* @RAM, i64 0, i64 %51
  %65 = bitcast i8* %64 to i32*
  store i32 9, i32* %65, align 1
  br label %66

66:                                               ; preds = %52, %63
  %67 = getelementptr inbounds [0 x i8], [0 x i8]* @RAM, i64 0, i64 %51
  %68 = bitcast i8* %67 to i32*
  store i32 20, i32* %68, align 1
  br label %69

69:                                               ; preds = %52, %66
  %70 = getelementptr inbounds [0 x i8], [0 x i8]* @RAM, i64 0, i64 %51
  %71 = bitcast i8* %70 to i32*
  store i32 30, i32* %71, align 1
  br label %72

72:                                               ; preds = %52, %69
  %73 = getelementptr inbounds [0 x i8], [0 x i8]* @RAM, i64 0, i64 %51
  %74 = bitcast i8* %73 to i32*
  store i32 55, i32* %74, align 1
  br label %75

75:                                               ; preds = %52, %72
  %76 = getelementptr inbounds [0 x i8], [0 x i8]* @RAM, i64 0, i64 %51
  %77 = bitcast i8* %76 to i32*
  store i32 99, i32* %77, align 1
  br label %78

78:                                               ; preds = %52, %75
  %79 = getelementptr inbounds [0 x i8], [0 x i8]* @RAM, i64 0, i64 %51
  %80 = bitcast i8* %79 to i32*
  store i32 100, i32* %80, align 1
  br label %81

81:                                               ; preds = %41, %78
  %82 = phi i64 [ %62, %78 ], [ %50, %41 ]
  %83 = phi i64 [ 5368709120, %78 ], [ %2, %41 ]
  %84 = getelementptr inbounds [0 x i8], [0 x i8]* @RAM, i64 0, i64 %51
  %85 = bitcast i8* %84 to i32*
  store i32 22, i32* %85, align 1
  store i64 %83, i64* %rcx, align 8
  br label %86

86:                                               ; preds = %37, %81
  %87 = phi i64 [ %82, %81 ], [ 3735929054, %37 ]
  %88 = phi i64 [ 5368727067, %81 ], [ 0, %37 ]
  store i64 %87, i64* %rax, align 8
  ret i64 %88
}


```

When the `PointersHoisting` pass is applied before executing the `DSE` pass we obtain the following code, where the switch case has been completely removed because it has been deemed dead.

```
define dso_local i64 @F_0x1400045b0(i64* noalias nocapture nonnull align 8 dereferenceable(8) %0, i64* noalias nocapture nonnull readnone align 8 dereferenceable(8) %1, i64* noalias nocapture nonnull align 8 dereferenceable(8) %2, i64* noalias nocapture nonnull readnone align 8 dereferenceable(8) %3, i64* noalias nocapture nonnull readnone align 8 dereferenceable(8) %4, i64* noalias nocapture nonnull readnone align 8 dereferenceable(8) %5, i64* noalias nocapture nonnull readnone align 8 dereferenceable(8) %6, i64* noalias nocapture nonnull readonly align 8 dereferenceable(8) %7, i64* noalias nocapture nonnull readnone align 8 dereferenceable(8) %8, i64* noalias nocapture nonnull readnone align 8 dereferenceable(8) %9, i64* noalias nocapture nonnull readnone align 8 dereferenceable(8) %10, i64* noalias nocapture nonnull readnone align 8 dereferenceable(8) %11, i64* noalias nocapture nonnull readnone align 8 dereferenceable(8) %12, i64* noalias nocapture nonnull readnone align 8 dereferenceable(8) %13, i64* noalias nocapture nonnull readnone align 8 dereferenceable(8) %14, i64* noalias nocapture nonnull readnone align 8 dereferenceable(8) %15, i64* noalias nocapture nonnull readnone align 8 dereferenceable(8) %16, i64 %17, i64 %18, i64 %19) local_unnamed_addr {
  %21 = load i64, i64* %7, align 8
  %22 = load i64, i64* %2, align 8
  %23 = tail call { i32, i32, i32, i32 } asm "  xchgq  %rbx,${1:q}\0A  cpuid\0A  xchgq  %rbx,${1:q}", "={ax},=r,={cx},={dx},0,~{dirflag},~{fpsr},~{flags}"(i32 1)
  %24 = extractvalue { i32, i32, i32, i32 } %23, 0
  %25 = extractvalue { i32, i32, i32, i32 } %23, 1
  %26 = and i32 %24, 4080
  %27 = icmp eq i32 %26, 4064
  %28 = xor i32 %24, 32
  %29 = select i1 %27, i32 %28, i32 %24
  %30 = and i32 %25, 16777215
  %31 = add i32 %29, %30
  %32 = load i64, i64* bitcast (i8* getelementptr inbounds ([0 x i8], [0 x i8]* @GS, i64 0, i64 96) to i64*), align 1
  %33 = add i64 %32, 288
  %34 = getelementptr inbounds [0 x i8], [0 x i8]* @RAM, i64 0, i64 %33
  %35 = bitcast i8* %34 to i16*
  %36 = load i16, i16* %35, align 1
  %37 = zext i16 %36 to i32
  %38 = load i64, i64* bitcast (i8* getelementptr inbounds ([0 x i8], [0 x i8]* @RAM, i64 0, i64 5369231568) to i64*), align 1
  %39 = shl nuw nsw i32 %37, 7
  %40 = add i64 %38, 32
  %41 = getelementptr inbounds [0 x i8], [0 x i8]* @RAM, i64 0, i64 %40
  %42 = bitcast i8* %41 to i32*
  %43 = load i32, i32* %42, align 1
  %44 = xor i32 %43, 2480
  %45 = add i32 %31, %39
  %46 = trunc i64 %38 to i32
  %47 = add i64 %38, 88
  %48 = xor i32 %45, %46
  %49 = add i64 %21, 32
  %50 = getelementptr inbounds [0 x i8], [0 x i8]* @RAM, i64 0, i64 %49
  br label %51

  %52 = phi i64 [ %47, %20 ], [ %60, %59 ]
  %53 = phi i32 [ %44, %20 ], [ %61, %59 ]
  %54 = getelementptr inbounds [0 x i8], [0 x i8]* @RAM, i64 0, i64 %52
  %55 = bitcast i8* %54 to i32*
  %56 = load i32, i32* %55, align 1
  %57 = xor i32 %56, 30826
  %58 = icmp eq i32 %48, %57
  br i1 %58, label %63, label %59

  %60 = add i64 %52, 8
  %61 = add i32 %53, -1
  %62 = icmp eq i32 %53, 1
  br i1 %62, label %88, label %51

  %64 = add i64 %21, 40
  %65 = getelementptr inbounds [0 x i8], [0 x i8]* @RAM, i64 0, i64 %64
  %66 = bitcast i8* %65 to i32*
  %67 = load i32, i32* %66, align 1
  %68 = add i64 %21, 36
  %69 = getelementptr inbounds [0 x i8], [0 x i8]* @RAM, i64 0, i64 %68
  %70 = bitcast i8* %69 to i32*
  store i32 %67, i32* %70, align 1
  %71 = icmp ugt i32 %67, 5
  %72 = zext i32 %67 to i64
  br i1 %71, label %84, label %73

  %74 = shl nuw nsw i64 %72, 1
  %75 = and i64 %74, 4294967296
  %76 = sub nsw i64 %72, %75
  %77 = shl nsw i64 %76, 2
  %78 = add nsw i64 %77, 5368964976
  %79 = getelementptr inbounds [0 x i8], [0 x i8]* @RAM, i64 0, i64 %78
  %80 = bitcast i8* %79 to i32*
  %81 = load i32, i32* %80, align 1
  %82 = zext i32 %81 to i64
  %83 = add nuw nsw i64 %82, 5368709120
  br label %84

  %85 = phi i64 [ %83, %73 ], [ %72, %63 ]
  %86 = phi i64 [ 5368709120, %73 ], [ %22, %63 ]
  %87 = bitcast i8* %50 to i32*
  store i32 22, i32* %87, align 1
  store i64 %86, i64* %2, align 8
  br label %88

  %89 = phi i64 [ %85, %84 ], [ 3735929054, %59 ]
  %90 = phi i64 [ 5368727067, %84 ], [ 0, %59 ]
  store i64 %89, i64* %0, align 8
  ret i64 %90
}


```

[ConstantConcretization](#constantconcretization)
-------------------------------------------------

This pass falls under the category of the generic LLVM optimization passes that are useful when dealing with obfuscated code, but basically useless, at least in the current shape, in a standard compilation pipeline. In fact, it’s not uncommon to find obfuscated code relying on constants stored in data sections added during the protection phase.

As an example, on some versions of VMProtect, when the `Ultra` mode is used, the conditional branch computations involve dummy constants fetched from a data section. Or if we think about a virtualized jump table (e.g. generated by a switch in the original binary), we also have to deal with a set of constants fetched from a data section.

Hence the reason for having a custom pass that, during the execution of the pipeline, identifies potential constant data accesses and converts the associated memory `load` into an LLVM [constant](https://llvm.org/doxygen/classllvm_1_1ConstantInt.html) (or chain of constants). This process can be referred to as constant(s) concretization.

The pass is going to identify all the load memory accesses in the function and determine if they fall in the following categories:

*   A [constantexpr](https://llvm.org/doxygen/classllvm_1_1ConstantExpr.html) memory load that is using an address contained in one of the binary sections; this case is what you would hit when dealing with some kind of data-based obfuscation;
*   A symbolic memory load that is using as base an address contained in one of the binary sections and as index an expression that is constrained to a limited amount of values; this case is what you would hit when dealing with a jump table.

In both cases the user needs to provide a safe set of memory ranges that the pass can consider as `read-only`, otherwise the pass will restrict the concretization to addresses falling in `read-only` sections in the binary.

In the first case, the address is directly available and the associated value can be resolved simply parsing the binary.

In the second case the expression computing the symbolic memory access is sliced, the constraint(s) coming from the predecessor block(s) are harvested and Souper is queried in an incremental way (conceptually similar to the one used while solving the outgoing edges in a `VmBlock`) to obtain the set of addresses accessing the binary. Each address is then verified to be really laying in a binary section and the corresponding value is fetched. At this point we have a unique mapping between each address and its value, that we can turn into a selection cascade, illustrated in the following LLVM-IR snippet:

```
; Fetching the switch control value from [rsp + 40]
%2 = add i64 %rsp, 40
%3 = getelementptr inbounds [0 x i8], [0 x i8]* @RAM, i64 0, i64 %2
%4 = bitcast i8* %3 to i32*
%72 = load i32, i32* %4, align 1
; Computing the symbolic address
%84 = zext i32 %72 to i64
%85 = shl nuw nsw i64 %84, 1
%86 = and i64 %85, 4294967296
%87 = sub nsw i64 %84, %86
%88 = shl nsw i64 %87, 2
%89 = add nsw i64 %88, 5368964976
; Generated selection cascade
%90 = icmp eq i64 %89, 5368964988
%91 = icmp eq i64 %89, 5368964980
%92 = icmp eq i64 %89, 5368964984
%93 = icmp eq i64 %89, 5368964992
%94 = icmp eq i64 %89, 5368964996
%95 = select i1 %90, i64 2442748, i64 1465288
%96 = select i1 %91, i64 650651, i64 %95
%97 = select i1 %92, i64 2740242, i64 %96
%98 = select i1 %93, i64 1706770, i64 %97
%99 = select i1 %94, i64 1510355, i64 %98


```

The `%99` value will hold the proper constant based on the address computed by the `%89` value. The example above represents the lifted jump table shown in the next snippet, where you can notice the jump table base `0x14003E770` (`5368964976`) and the corresponding addresses and values:

```
.rdata:0x14003E770 dd 0x165BC8
.rdata:0x14003E774 dd 0x9ED9B
.rdata:0x14003E778 dd 0x29D012
.rdata:0x14003E77C dd 0x2545FC
.rdata:0x14003E780 dd 0x1A0B12
.rdata:0x14003E784 dd 0x170BD3


```

If we have a peek at the sliced jump condition implementing the virtualized switch case (below), this is how it looks after the `ConstantConcretization` pass has been scheduled in the pipeline and further `InstCombine` executions updated the selection cascade to compute the switch case addresses. Souper will therefore be able to identify the 6 possible outgoing edges, leading to the devirtualized switch case presented in the `PointersHoisting` section:

```
; Fetching the switch control value from [rsp + 40]
%2 = add i64 %rsp, 40
%3 = getelementptr inbounds [0 x i8], [0 x i8]* @RAM, i64 0, i64 %2
%4 = bitcast i8* %3 to i32*
%72 = load i32, i32* %4, align 1
; Computing the symbolic address
%84 = zext i32 %72 to i64
%85 = shl nuw nsw i64 %84, 1
%86 = and i64 %85, 4294967296
%87 = sub nsw i64 %84, %86
%88 = shl nsw i64 %87, 2
%89 = add nsw i64 %88, 5368964976
; Generated selection cascade
%90 = icmp eq i64 %89, 5368964988
%91 = icmp eq i64 %89, 5368964980
%92 = icmp eq i64 %89, 5368964984
%93 = icmp eq i64 %89, 5368964992
%94 = icmp eq i64 %89, 5368964996
%95 = select i1 %90, i64 5371151872, i64 5370415894
%96 = select i1 %91, i64 5369359775, i64 %95
%97 = select i1 %92, i64 5371449366, i64 %96
%98 = select i1 %93, i64 5370174412, i64 %97
%99 = select i1 %94, i64 5370219479, i64 %98
%100 = call i64 @HelperKeepPC(i64 %99) #15


```

It is well known that all the virtualization-based protectors support only a subset of the targeted ISA. Thus, when an unsupported instruction is found, an exit from the virtual machine is executed (context switching to the host code), running the unsupported instruction(s) and re-entering the virtual machine (context switching back to the virtualized code).

The [UnsupportedInstructionsLiftingToLLVM](https://github.com/LLVMParty/UnsupportedInstructionsLiftingToLLVM) proof-of-concept is an attempt to lift the unsupported instructions to LLVM-IR, generating an [InlineAsm](https://llvm.org/doxygen/classllvm_1_1InlineAsm.html) instruction configured with the set of clobbering constraints and (ex|im)plicitly accessed registers. An execution context structure representing the general purpose registers is employed during the lifting to feed the inline assembly call instruction with the loaded registers, and to store the updated registers after the inline assembly execution.

This approach guarantees a smooth connection between two virtualized `VmStubs` and an intermediate sequence of unsupported instructions, enabling some of the LLVM optimizations and a better registers allocation during the recompilation phase.

An example of the lifted unsupported instruction `rdtsc` follows:

```
define void @Unsupported_rdtsc(%ContextTy* nocapture %0) local_unnamed_addr #0 {
  %2 = tail call %IAOutTy.0 asm sideeffect inteldialect "rdtsc", "={eax},={edx}"() #1
  %3 = bitcast %ContextTy* %0 to i32*
  %4 = extractvalue %IAOutTy.0 %2, 0
  store i32 %4, i32* %3, align 4
  %5 = getelementptr inbounds %ContextTy, %ContextTy* %0, i64 0, i32 3, i32 0
  %6 = bitcast %RegisterW* %5 to i32*
  %7 = extractvalue %IAOutTy.0 %2, 1
  store i32 %7, i32* %6, align 4
  ret void
}


```

An example of the lifted unsupported instruction `cpuid` follows:

```
define void @Unsupported_cpuid(%ContextTy* nocapture %0) local_unnamed_addr #0 {
  %2 = bitcast %ContextTy* %0 to i32*
  %3 = load i32, i32* %2, align 4
  %4 = getelementptr inbounds %ContextTy, %ContextTy* %0, i64 0, i32 2, i32 0
  %5 = bitcast %RegisterW* %4 to i32*
  %6 = load i32, i32* %5, align 4
  %7 = tail call %IAOutTy asm sideeffect inteldialect "cpuid", "={eax},={ecx},={edx},={ebx},{eax},{ecx}"(i32 %3, i32 %6) #1
  %8 = extractvalue %IAOutTy %7, 0
  store i32 %8, i32* %2, align 4
  %9 = extractvalue %IAOutTy %7, 1
  store i32 %9, i32* %5, align 4
  %10 = getelementptr inbounds %ContextTy, %ContextTy* %0, i64 0, i32 3, i32 0
  %11 = bitcast %RegisterW* %10 to i32*
  %12 = extractvalue %IAOutTy %7, 2
  store i32 %12, i32* %11, align 4
  %13 = getelementptr inbounds %ContextTy, %ContextTy* %0, i64 0, i32 1, i32 0
  %14 = bitcast %RegisterW* %13 to i32*
  %15 = extractvalue %IAOutTy %7, 3
  store i32 %15, i32* %14, align 4
  ret void
}


```

I haven’t really explored the recompilation in depth so far, because my main objective was to obtain readable LLVM-IR code, but some considerations follow:

*   If the goal is being able to execute, and eventually decompile, the recovered code, then compiling the devirtualized function using the layer of indirection offered by the general purpose register pointers is a valid way to do so. It is conceptually similar to the kind of indirection used by Remill with its own `State` structure. SATURN employs this technique when the stack slots and arguments recovery cannot be applied.
*   If the goal is to achieve a 1:1 register allocation, then things get more complicated because one can’t simply map all the general purpose register pointers to the hardware registers hoping for no side-effect to manifest.

The major issue to deal with when attempting a 1:1 mapping is related to how the recompilation may unexpectedly change the stack layout. This could happen if, during the [register allocation](https://en.wikipedia.org/wiki/Register_allocation) phase, some spilling slot is allocated on the stack. If these additional spilling+reloading semantics are not adequately handled, some pointers used by the function may access unforeseen stack slots with disastrous results.

The following log files contain the output of the PoC tool executed on functions showcasing different code constructs (e.g. loop, jump table) and accessing different data structures (e.g. GS segment, DS segment, KUSER_SHARED_DATA structure):

*   [0x140001d10@DevirtualizeMe1](https://github.com/LLVMParty/TicklingVMProtect/blob/master/Results/DevirtualizeMe1Func1.ll): calling `KERNEL32.dll::GetTickCount64` and literally included as nostalgia kicked in;
*   [0x140001e60@DevirtualizeMe2](https://github.com/LLVMParty/TicklingVMProtect/blob/master/Results/DevirtualizeMe2Func2.ll): executing an unsupported `cpuid` and with some nicely recovered `llvm.fshl.i64` intrinsic calls used as rotations;
*   [0x140001d20@DevirtualizeMe2](https://github.com/LLVMParty/TicklingVMProtect/blob/master/Results/DevirtualizeMe2Func1.ll): calling `ADVAPI32.dll::GetUserNameW` and with a nicely recovered `llvm.bswap.i64` intrinsic call;
*   [0x13001334@EMP](https://github.com/LLVMParty/TicklingVMProtect/blob/master/Results/RandomEMPFunc2.ll): DllEntryPoint, calling another internal function (intra-call);
*   [0x1301d000@EMP](https://github.com/LLVMParty/TicklingVMProtect/blob/master/Results/RandomEMPFunc3.ll): calling `KERNEL32.dll::LoadLibraryA`, `KERNEL32.dll::GetProcAddress`, calling other internal functions (intra-calls), executing several unsupported `cpuid` instructions;
*   [0x130209c0@EMP](https://github.com/LLVMParty/TicklingVMProtect/blob/master/Results/RandomEMPFunc1.ll): accessing `KUSER_SHARED_DATA` and with nicely synthesized conditional jumps;
*   [0x1400044c0@Switches64](https://github.com/LLVMParty/TicklingVMProtect/blob/master/Results/Switches64Func1.ll): executing the `CPUID` handler and devirtualized with `PointersHoisting` disabled to preserve the switch case.

Searching for the `@F_` pattern in your favourite text editor will bring you directly to each devirtualized `VmStub`, immediately preceded by the textual representation of the recovered CFG.

I apologize for the length of the series, but I didn’t want to discard bits of information that could possibly help others approaching LLVM as a deobfuscation framework, especially knowing that, at this time, several parties are currently working on their own LLVM-based solution. I felt like showcasing its effectiveness and limitations on a well-known obfuscator was a valid way to dive through most of the details. Please note that the process described in the posts is just one of the many possible ways to approach the problem, and by no means the best way.

The [source code](https://github.com/LLVMParty/TicklingVMProtect) of the proof-of-concept should be considered an experimentation playground, with everything that involves (e.g. bugs, unhandled edge cases, non production-ready quality). As a matter of fact, some of the components are barely sketched to let me focus on improving the LLVM optimization pipeline. In the future I’ll try to find the time to polish most of it, but in the meantime I hope it can at least serve as a reference to better understand the explained concepts.

Feel free to reach out with doubts, questions or even flaws you may have found in the process, I’ll be more than happy to allocate some time to discuss them.

I’d like to thank:

*   Peter, for introducing me to LLVM and working on SATURN together.
*   mrexodia and mrphrazer, for the in-depth review of the posts.
*   justmusjle, for enhancing the colors used by the diagrams.
*   Secret Club, for their suggestions and series hosting.

See you at the next LLVM adventure!