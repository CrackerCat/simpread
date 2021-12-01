> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [secret.club](https://secret.club/2021/09/08/vmprotect-llvm-lifting-2.html)

> This post will introduce the concepts of expression slicing and partial CFG, combining them to implem......

This post will introduce the concepts of _expression slicing_ and _partial CFG_, combining them to implement an SMT-driven algorithm to explore the virtualized CFG. Finally, some words will be spent on introducing the LLVM optimization pipeline, its configuration and its limitations.

Slicing a symbolic expression to be able to evaluate it, throw it at an SMT solver or match it against some pattern is something extremely common in all symbolic reasoning tools. Luckily for us this capability is trivial to implement with yet another C++ helper function. This technique has been referred to as _Poor man’s slicer_ in the SATURN paper, hence the title of the section.

In the VMProtect context we are mainly interested in slicing one expression: the next program counter. We want to do that either while exploring the single `VmBlocks` (that, once connected, form a `VmStub`) or while exploring the `VmStubs` (that, once connected, form a `VmFunction`). The following C++ code is meant to keep only the computations related to the final value of the virtual instruction pointer at the end of a `VmBlock` or `VmStub`:

```
extern "C"
size_t HelperSlicePC(
  size_t rax, size_t rbx, size_t rcx, size_t rdx, size_t rsi,
  size_t rdi, size_t rbp, size_t rsp, size_t r8, size_t r9,
  size_t r10, size_t r11, size_t r12, size_t r13, size_t r14,
  size_t r15, size_t flags,
  size_t KEY_STUB, size_t RET_ADDR, size_t REL_ADDR)
{
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
  vip = HelperStub(
    rax, rbx, rcx, rdx, rsi, rdi,
    rbp, rsp, r8, r9, r10, r11,
    r12, r13, r14, r15, flags,
    KEY_STUB, RET_ADDR, REL_ADDR,
    vsp, vip, vmregs, slots);
  // Return the sliced program counter
  return vip;
}


```

The acute observer will notice that the function definition is basically identical to the `HelperFunction` definition given before, with the fundamental difference that the arguments are passed by value and therefore useful if related to the computation of the sliced expression, but with their liveness scope ending at the end of the function, which guarantees that there won’t be store operations to the host context that could possibly bloat the code.

The steps to use the above helper function are:

*   The `HelperSlicePC` is cloned into a new throwaway function;
*   The call to the `HelperStub` function is swapped with a call to the `VmBlock` or `VmStub` of which we want to slice the final instruction pointer;
*   The called function is forcefully inlined into the `HelperSlicePC` function;
*   The optimization pipeline is executed on the cloned `HelperSlicePC` function resulting in the slicing of the final instruction pointer expression as a side-effect of the optimizations.

The following LLVM-IR snippet shows the idea in action, resulting in the final optimized function where the condition and edges of the conditional branch are clearly visible.

```
define dso_local i64 @HelperSlicePC(i64 %rax, i64 %rbx, i64 %rcx, i64 %rdx, i64 %rsi, i64 %rdi, i64 %rbp, i64 %rsp, i64 %r8, i64 %r9, i64 %r10, i64 %r11, i64 %r12, i64 %r13, i64 %r14, i64 %r15, i64 %flags, i64 %KEY_STUB, i64 %RET_ADDR, i64 %REL_ADDR) #5 {
  %1 = call { i32, i32, i32, i32 } asm "  xchgq  %rbx,${1:q}\0A  cpuid\0A  xchgq  %rbx,${1:q}", "={ax},=r,={cx},={dx},0,~{dirflag},~{fpsr},~{flags}"(i32 1) #4, !srcloc !9
  %2 = extractvalue { i32, i32, i32, i32 } %1, 0
  %3 = and i32 %2, 4080
  %4 = icmp eq i32 %3, 4064
  %5 = select i1 %4, i64 5371013457, i64 5371013227
  ret i64 %5
}


```

In the following section we’ll see how variations of this technique are used to explore the virtualized control flow graph, solve the conditional branches, and recover the switch cases.

The exploration of a virtualized control flow graph can be done in different ways and usually protectors like VMProtect or Themida show a distinctive shape that can be pattern-matched with ease, simplified and parsed to obtain the outgoing edges of a conditional block.

The logic used by different VMProtect conditional jump versions has been detailed in the past, so in this section we are going to delve into an SMT-driven algorithm based on the incremental construction of the explored control flow graph and specifically built on top of the slicing logic explained in the previous section.

Given the generic nature of the detailed algorithm, nothing stops it from being used on other protectors. The usual catch is obviously caused by protections embedding hard to solve constraints that may hinder the automated solving phase, but the construction and propagation of the partial CFG constraints and expressions could still be useful in practice to pull out less automated exploration algorithms, or to identify and simplify anti-dynamic symbolic execution tricks (e.g. dummy loops leading to path explosion that could be simplified by LLVM’s loop optimization passes or custom user passes).

[Partial CFG](#partial-cfg)
---------------------------

A partial control flow graph is a control flow graph built connecting the currently explored basic blocks given the known edges between them. The idea behind building it, is that each time that we explore a new basic block, we gather new outgoing edges that could lead to new unexplored basic blocks, or even to known basic blocks. Every new edge between two blocks is therefore adding information to the entire control flow graph and we could actually propagate new useful constraints and values to enable stronger optimizations, possibly easing the solving of the conditional branches or even changing a known branch from unconditional to conditional.

Let’s look at two motivating examples of why building a partial CFG may be a good idea to be able to replicate the kind of reasoning usually implemented by symbolic execution tools, with the addition of useful built-in LLVM passes.

### [Motivating example #1](#motivating-example-1)

Consider the following partial control flow graph, where blue represents the `VmBlock` that has just been processed, orange the unprocessed `VmBlock` and purple the `VmBlock` of interest for the example.

![](https://secret.club/assets/vmp2llvm/diagram1.svg)

Let’s assume we just solved the outgoing edges for the basic block `A`, obtaining two connections leading to the new basic blocks `B` and `C`. Now assume that we sliced the branch condition of the sole basic block `B`, obtaining an access into a constant array with a 64 bits symbolic index. Enumerating all the valid indices may be a non-trivial task, so we may want to restrict the search using known constraints on the symbolic index that, if present, are most likely going to come from the chain(s) of predecessor(s) of the basic block `B`.

To draw a symbolic execution parallel, this is the case where we want to collect the path constraints from a certain number of predecessors (e.g. we may want to incrementally harvest the constraints, because sometimes the needed constraint is locally near to the basic block we are solving) and chain them to be fed to an SMT solver to execute a successful enumeration of the valid indices.

Tools like [Souper](https://github.com/google/souper) automatically harvest the set of path constraints while slicing an expression, so building the partial control flow graph and feeding it to Souper may be sufficient for the task. Additionally, with the LLVM API to walk the predecessors of a basic block it’s also quite easy to obtain the set of needed constraints and, when available, we may also take advantage of known-to-be-true conditions provided by the [llvm.assume](https://llvm.org/docs/LangRef.html#llvm-assume-intrinsic) intrinsic.

### [Motivating example #2](#motivating-example-2)

Consider the following partial control flow graph, where blue represents the `VmBlock` that has just been processed, orange the unprocessed `VmBlocks`, purple the `VmBlock` of interest for the example, dashed red arrows the edges of interest for the example and the solid green arrow an edge that has just been processed.

![](https://secret.club/assets/vmp2llvm/diagram0.svg)

Let’s assume we just solved the outgoing edges for the basic block `E`, obtaining two connections leading to a new block `G` and a known block `B`. In this case we know that we detected a jump to the previously visited block `B` (edge in green), which is basically forming a loop chain (`B` → `C` → `E` → `B`) and we know that starting from `B` we can reach two edges (`B` → `C` and `D` → `F`, marked in dashed red) that are currently known as unconditional, but that, given the newly obtained edge `E` → `B`, may not be anymore and therefore will need to be proved again. Building a new partial control flow graph including all the newly discovered basic block connections and slicing the branch of the blocks `B` and `D` may now show them as conditional.

As a real world case, when dealing with concolic execution approaches, the one mentioned above is the usual pattern that arises with index-based loops, starting with a known concrete index and running till the index reaches an upper or lower bound `N`. During the first `N-1` executions the tool would take the same path and only at the iteration `N` the other path would be explored. That’s the reason why concolic and symbolic execution tools attempt to build heuristics or use techniques like state-merging to avoid running into [path explosion](https://en.wikipedia.org/wiki/Symbolic_execution#Path_explosion) issues (or at best executing the loop `N` times).

Building the partial CFG with LLVM instead, would mark the loop back edge as unconditional the first time, but building it again, including the knowledge of the newly discovered back edge, would immediately reveal the loop pattern. The outcome is that LLVM would now be able to apply its loop analysis passes, the user would be able to use the API to build ad-hoc [LoopPass](https://llvm.org/docs/WritingAnLLVMPass.html#the-looppass-class) passes to handle special obfuscation applied to the loop components (e.g. encoded loop variant/invariant) or the SMT solvers would be able to treat newly created [Phi](https://en.wikipedia.org/wiki/Static_single_assignment_form) nodes at the merge points as symbolic variables.

The following LLVM-IR snippet shows the sliced partial control flow graphs obtained during the exploration of the virtualized assembly snippet presented below.

```
start:
  mov rax, 2000
  mov rbx, 0
loop:
  inc rbx
  cmp rbx, rax
  jne loop


```

The original assembly snippet.

```
define dso_local i64 @FirstSlice(i64 %rax, i64 %rbx, i64 %rcx, i64 %rdx, i64 %rsi, i64 %rdi, i64 %rbp, i64 %rsp, i64 %r8, i64 %r9, i64 %r10, i64 %r11, i64 %r12, i64 %r13, i64 %r14, i64 %r15, i64 %flags, i64 %KEY_STUB, i64 %RET_ADDR, i64 %REL_ADDR) {
  %1 = call i64 @HelperKeepPC(i64 5369464257)
  ret i64 %1
}


```

The first partial CFG obtained during the exploration phase

```
define dso_local i64 @SecondSlice(i64 %rax, i64 %rbx, i64 %rcx, i64 %rdx, i64 %rsi, i64 %rdi, i64 %rbp, i64 %rsp, i64 %r8, i64 %r9, i64 %r10, i64 %r11, i64 %r12, i64 %r13, i64 %r14, i64 %r15, i64 %flags, i64 %KEY_STUB, i64 %RET_ADDR, i64 %REL_ADDR) {
  br label %1

1:                                                ; preds = %1, %0
  %2 = phi i64 [ 0, %0 ], [ %3, %1 ]
  %3 = add i64 %2, 1
  %4 = icmp eq i64 %2, 1999
  %5 = select i1 %4, i64 5369183207, i64 5369464257
  %6 = call i64 @HelperKeepPC(i64 %5)
  %7 = icmp eq i64 %6, 5369464257
  br i1 %7, label %1, label %8

8:                                                ; preds = %1
  ret i64 233496237
}


```

The second partial CFG obtained during the exploration phase. The block `8` is returning the dummy `0xdeaddead` (`233496237`) value, meaning that the `VmBlock` instructions haven’t been lifted yet.

```
define dso_local i64 @F_0x14000101f_NoLoopOpt(i64* noalias nonnull align 8 dereferenceable(8) %rax, i64* noalias nonnull align 8 dereferenceable(8) %rbx, i64* noalias nonnull align 8 dereferenceable(8) %rcx, i64* noalias nonnull align 8 dereferenceable(8) %rdx, i64* noalias nonnull align 8 dereferenceable(8) %rsi, i64* noalias nonnull align 8 dereferenceable(8) %rdi, i64* noalias nonnull align 8 dereferenceable(8) %rbp, i64* noalias nonnull align 8 dereferenceable(8) %rsp, i64* noalias nonnull align 8 dereferenceable(8) %r8, i64* noalias nonnull align 8 dereferenceable(8) %r9, i64* noalias nonnull align 8 dereferenceable(8) %r10, i64* noalias nonnull align 8 dereferenceable(8) %r11, i64* noalias nonnull align 8 dereferenceable(8) %r12, i64* noalias nonnull align 8 dereferenceable(8) %r13, i64* noalias nonnull align 8 dereferenceable(8) %r14, i64* noalias nonnull align 8 dereferenceable(8) %r15, i64* noalias nonnull align 8 dereferenceable(8) %flags, i64 %KEY_STUB, i64 %RET_ADDR, i64 %REL_ADDR) {
  br label %1

1:                                                ; preds = %1, %0
  %2 = phi i64 [ 0, %0 ], [ %3, %1 ]
  %3 = add i64 %2, 1
  %4 = icmp eq i64 %2, 1999
  br i1 %4, label %5, label %1

5:                                                ; preds = %1
  store i64 %3, i64* %rbx, align 8
  store i64 2000, i64* %rax, align 8
  ret i64 5368713281
}


```

The final CFG obtained at the completion of the exploration phase.

```
define dso_local i64 @F_0x14000101f_WithLoopOpt(i64* noalias nonnull align 8 dereferenceable(8) %rax, i64* noalias nonnull align 8 dereferenceable(8) %rbx, i64* noalias nonnull align 8 dereferenceable(8) %rcx, i64* noalias nonnull align 8 dereferenceable(8) %rdx, i64* noalias nonnull align 8 dereferenceable(8) %rsi, i64* noalias nonnull align 8 dereferenceable(8) %rdi, i64* noalias nonnull align 8 dereferenceable(8) %rbp, i64* noalias nonnull align 8 dereferenceable(8) %rsp, i64* noalias nonnull align 8 dereferenceable(8) %r8, i64* noalias nonnull align 8 dereferenceable(8) %r9, i64* noalias nonnull align 8 dereferenceable(8) %r10, i64* noalias nonnull align 8 dereferenceable(8) %r11, i64* noalias nonnull align 8 dereferenceable(8) %r12, i64* noalias nonnull align 8 dereferenceable(8) %r13, i64* noalias nonnull align 8 dereferenceable(8) %r14, i64* noalias nonnull align 8 dereferenceable(8) %r15, i64* noalias nonnull align 8 dereferenceable(8) %flags, i64 %KEY_STUB, i64 %RET_ADDR, i64 %REL_ADDR) {
  store i64 2000, i64* %rbx, align 8
  store i64 2000, i64* %rax, align 8
  ret i64 5368713281
}


```

The loop-optimized final CFG obtained at the completion of the exploration phase.

The `FirstSlice` function shows that a single unconditional branch has been detected, identifying the bytecode address **0x1400B85C1** (**5369464257**), this is because there’s no knowledge of the back edge and the comparison would be **cmp 1, 2000**. The `SecondSlice` function instead shows that a conditional branch has been detected selecting between the bytecode addresses **0x140073BE7** (**5369183207**) and **0x1400B85C1** (**5369464257**). The comparison is now done with a symbolic [PHINode](https://llvm.org/doxygen/classllvm_1_1PHINode.html). The `F_0x14000101f_WithLoopOpt` and `F_0x14000101f_NoLoopOpt` functions show the fully devirtualized code with and without loop optimizations applied.

[Pseudocode](#pseudocode)
-------------------------

Given the knowledge obtained from the motivating examples, the pseudocode for the automated partial CFG driven exploration is the following:

1.  We initialize the algorithm creating:
    
    *   A `stack` of addresses of `VmBlocks` to explore, referred to as `Worklist`;
    *   A `set` of addresses of explored `VmBlocks`, referred to as `Explored`;
    *   A `set` of addresses of `VmBlocks` to reprove, referred to as `Reprove`;
    *   A `map` of known edges between the `VmBlocks`, referred to as `Edges`.
2.  We push the address of the entry `VmBlock` into the `Worklist`;
3.  We fetch the address of a `VmBlock` to explore, we lift it to LLVM-IR if met for the first time, we build the partial CFG using the knowledge from the `Edges` map and we slice the branch condition of the current `VmBlock`. Finally we feed the branch condition to Souper, which will process the expression harvesting the needed constraints and converting it to an SMT query. We can then send the query to an SMT solver, asking for the valid solutions, incrementally rejecting the known solutions up to some limit (worst case) or till all the solutions have been found.
4.  Once we obtained the outgoing edges for the current `VmBlock`, we can proceed with updating the maps and sets:
    
    *   We verify if each solved edge is leading to a known `VmBlock`; if it is, we verify if this connection was previously known. If unknown, it means we found a new predecessor for a known `VmBlock` and we proceed with adding the addresses of all the `VmBlocks` reachable by the known `VmBlock` to the `Reprove` set and removing them from the `Explored` set; to speed things up, we can eventually skip each `VmBlock` known to be firmly unconditional;
    *   We update the `Edges` map with the newly solved edges.
5.  At this point we check if the `Worklist` is empty. If it isn’t, we jump back to step **3**. If it is, we populate it with all the addresses in the `Reprove` set, clearing it in the process and jumping back to step **3**. If also the `Reprove` set is empty, it means we explored the whole CFG and eventually reproved all the `VmBlocks` that obtained new predecessors during the exploration phase.

As mentioned at the start of the section, there are many ways to explore a virtualized CFG and using an SMT-driven solution may generalize most of the steps. Obviously, it brings its own set of issues (e.g. hard to solve constraints), so one could eventually fall back to the pattern matching based solution at need. As expected, the pattern matching based solution would also blindly explore unreachable paths at times, so a mixed solution could really offer the best CFG coverage.

The pseudocode presented in this section is a simplified version of the partial CFG based exploration algorithm used by SATURN at this point in time, streamlined from a set of reasonings that are unnecessary while exploring a CFG virtualized by VMProtect.

So far we hinted at the underlying usage of LLVM’s optimization and analysis passes multiple times through the sections, so we can finally take a look at: how they fit in, their configuration and their limitations.

[Managing the pipeline](#managing-the-pipeline)
-----------------------------------------------

Running the whole `-O3` pipeline may not always be the best idea, because we may want to use only a subset of passes, instead of wasting cycles on passes that we know a priori don’t have any effect on the lifted LLVM-IR code. Additionally, by default, LLVM is providing a chain of optimizations which is executed once, is meant to optimize non-obfuscated code and should be as efficient as possible.

Although, in our case, we have different needs and want to be able to:

*   Add some custom passes to tackle context-specific problems and do so at precise points in the pipeline to obtain the best possible output, while avoiding [phase ordering](https://ieeexplore.ieee.org/document/1611550) issues;
*   Iterate the optimization pipeline more than once, ideally until our custom passes can’t apply any more changes to the IR code;
*   Be able to pass custom flags to the pipeline to toggle some passes at will and eventually feed them with information obtained from the binary (e.g. access to the binary sections).

LLVM provides a [FunctionPassManager](https://llvm.org/doxygen/classllvm_1_1legacy_1_1FunctionPassManager.html) class to craft our own pipeline, using LLVM’s passes and custom passes. The following C++ snippet shows how we can add a mix of passes that will be executed in order until there won’t be any more changes or until a threshold will be reached:

```
void optimizeFunction(llvm::Function *F, OptimizationGuide &G) {
  // Fetch the Module
  auto *M = F->getParent();
  // Create the function pass manager
  llvm::legacy::FunctionPassManager FPM(M);
  // Initialize the pipeline
  llvm::PassManagerBuilder PMB;
  PMB.OptLevel = 3;
  PMB.SizeLevel = 2;
  PMB.RerollLoops = false;
  PMB.SLPVectorize = false;
  PMB.LoopVectorize = false;
  PMB.Inliner = createFunctionInliningPass();
  // Add the alias analysis passes
  FPM.add(createCFLSteensAAWrapperPass());
  FPM.add(createCFLAndersAAWrapperPass());
  FPM.add(createTypeBasedAAWrapperPass());
  FPM.add(createScopedNoAliasAAWrapperPass());
  // Add some useful LLVM passes
  FPM.add(createCFGSimplificationPass());
  FPM.add(createSROAPass());
  FPM.add(createEarlyCSEPass());
  // Add a custom pass here
  if (G.RunCustomPass1)
    FPM.add(createCustomPass1(G));
  FPM.add(createInstructionCombiningPass());
  FPM.add(createCFGSimplificationPass());
  // Add a custom pass here
  if (G.RunCustomPass2)
    FPM.add(createCustomPass2(G));
  FPM.add(createGVNHoistPass());
  FPM.add(createGVNSinkPass());
  FPM.add(createDeadStoreEliminationPass());
  FPM.add(createInstructionCombiningPass());
  FPM.add(createCFGSimplificationPass());
  // Execute the pipeline
  size_t minInsCount = F->getInstructionCount();
  size_t pipExeCount = 0;
  FPM.doInitialization();
  do {
    // Reset the IR changed flag
    G.HasChanged = false;
    // Run the optimizations
    FPM.run(*F);
    // Check if the function changed
    size_t curInsCount = F->getInstructionCount();
    if (curInsCount < minInsCount) {
      minInsCount = curInsCount;
      G.HasChanged |= true;
    }
    // Increment the execution count
    pipExeCount++;
  } while (G.HasChanged && pipExeCount < 5);
  FPM.doFinalization();
}


```

The `OptimizationGuide` structure can be used to pass information to the custom passes and control the execution of the pipeline.

[Configuration](#configuration)
-------------------------------

As previously stated, the LLVM default pipeline is meant to be as efficient as possible, therefore it’s configured with a tradeoff between efficiency and efficacy in mind. While devirtualizing big functions it’s not uncommon to see the effects of the stricter configurations employed by default. But an [example](https://godbolt.org/z/7Throjsa5) is worth a thousand words.

In the Godbolt UI we can see on the left a snippet of LLVM-IR code that is storing **i32** values at increasing indices of a global array named **arr**. The `store` at line 96, writing the value **91** at **arr[1]**, is a bit special because it is fully overwriting the `store` at line 6, writing the value **1** at **arr[1]**. If we look at the upper right result, we see that the `DSE` pass was applied, but somehow it didn’t do its job of removing the dead `store` at line 6. If we look at the bottom right result instead, we see that the `DSE` pass managed to achieve its goal and successfully killed the dead `store` at line 6. The reason for the difference is entirely associated to a conservative configuration of the `DSE` pass, which by default (at the time of writing), is walking up to 90 [MemorySSA](https://llvm.org/docs/MemorySSA.html) definitions before deciding that a **store** is not killing another post-dominated `store`. Setting the [MemorySSAUpwardsStepLimit](https://llvm.org/doxygen/DeadStoreElimination_8cpp.html#aa9d7ec729de7347f5dedc03860919917) to a higher value (e.g. 100 in the example) is definitely something that we want to do while deobfuscating some code.

Each pass that we are going to add to the custom pipeline is going to have configurations that may be giving suboptimal deobfuscation results, so it’s a good idea to check their C++ implementation and figure out if tweaking some of the options may improve the output.

[Limitations](#limitations)
---------------------------

When tweaking some configurations is not giving the expected results, we may have to dig deeper into the implementation of a pass to understand if something is hindering its job, or roll up our sleeves and [develop](https://llvm.org/docs/WritingAnLLVMPass.html) a custom LLVM pass. Some examples on why digging into a pass implementation may lead to fruitful improvements follow.

### [IsGuaranteedLoopInvariant (DSE, MSSA)](#isguaranteedloopinvariant-dse-mssa)

While looking at some devirtualized code, I noticed some clearly-dead stores that weren’t removed by the `DSE` pass, even though the tweaked configurations were enabled. A minimal example of the problem, its explanation and solution are provided in the following diffs: [D96979](https://reviews.llvm.org/D96979), [D97155](https://reviews.llvm.org/D97155). The bottom line is that the `IsGuarenteedLoopInvariant` function used by the `DSE` and `MSSA` passes was not using the safe assumption that a pointer computed in the entry block is, by design, guaranteed to be loop invariant as the entry block of a [Function](https://llvm.org/doxygen/classllvm_1_1Function.html) is guaranteed to have no predecessors and not to be part of a loop.

### [GetPointerBaseWithConstantOffset (DSE)](#getpointerbasewithconstantoffset-dse)

While looking at some devirtualized code that was accessing memory slots of different sizes, I noticed some clearly-dead stores that weren’t removed by the `DSE` pass, even though the tweaked configurations were enabled. A minimal example of the problem, its explanation and solution are provided in the following diff: [D97676](https://reviews.llvm.org/D97676). The bottom line is that while computing the partially overlapping memory stores, the `DSE` was considering only memory slots with the same base address, ignoring fully overlapping stores offsetted between each other. The solution is making use of another patch which is providing information about the offsets of the memory slots: [D93529](https://reviews.llvm.org/D93529).

### [Shift-Select folding (InstCombine)](#shift-select-folding-instcombine)

And obviously there is no two without three! Nah, just kidding, a patch I wanted to get accepted to simplify one of the recurring patterns in the computation of the VMProtect conditional branches has been on pause because [InstCombine](https://llvm.org/doxygen/InstructionCombining_8cpp.html) is an extremely complex piece of code and additions to it, especially if related to uncommon patterns, are unwelcome and seen as possibly bloating and slowing down the entire pipeline. Additional information on the pattern and the reasons that hinder its folding are available in the following differential: [D84664](https://reviews.llvm.org/D84664). Nothing stops us from maintaining our own version of `InstCombine` as a custom pass, with ad-hoc patterns specifically selected for the obfuscation under analysis.

In [Part 3](https://secret.club/2021/09/08/vmprotect-llvm-lifting-3.html) we’ll have a look at a list of custom passes necessary to reach a superior output quality. Then, some words will be spent on the handling of the unsupported instructions and on the recompilation process. Last but not least, the output of 6 devirtualized functions, with varying code constructs, will be shown.