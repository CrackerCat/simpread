> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-282176.htm)

> [原创] 从 Clang 到 Pass 加载与执行的流程

一、gdb 动调 clang
==============

文章开始之前，先通过 gd 动调 clang 来看一看整个过程是什么样的。

开始调试

```
gdb clang

```

![](https://bbs.kanxue.com/upload/attach/202406/985561_BCK2QSWE2K3P8G6.webp)

我这里调试的是 release 版本的 clang（不想再编译了），显然提示了找不到 clang 的调试符号。我建议使用 debug 版的 clang 来调试，不然就看不到断点处的源码了，这对调试还是有影响的（有些函数下不了断点，不知道是不是这个原因）。

直接把断点下在 main 函数处

```
gdb main

```

重要的来了！clang 在执行过程中，会 fork 子进程，如果直接进行跟踪会出现如下结果：  
![](https://bbs.kanxue.com/upload/attach/202406/985561_QWMTXDW4QKG4Z2M.webp)

因此我们需要通过如下指令在 gdb 中设置跟踪子进程：

```
set follow-fork-mode child

```

然后启动 clang 并运行我们的 Pass，指令如下：

```
run -mllvm -mypass ~/Desktop/test.cpp -o ~/Desktop/test_debug_clang.ll

```

![](https://bbs.kanxue.com/upload/attach/202406/985561_J24ZTVGV863R6K6.webp)

此时我们还需要对自己的 Pass 下断点：

```
b MyPass::runOnFunction

```

继续运行直到命中我们下的 Pass 断点。

![](https://bbs.kanxue.com/upload/attach/202406/985561_B3UV4PZP8FZ8JJH.webp)

成功断在了预期位置处，此时我们查看函数调用堆栈，指令为`bt`。

![](https://bbs.kanxue.com/upload/attach/202406/985561_GM7QNZTHTVNATP6.webp)

从上面的函数调用堆栈可以明了的看出 Pass 从被加载到被执行的整个过程（实际上，它还是缺少了一些函数）。

接下来我们结合源码来仔细分析一下这个流程。

二、从 clang 到 Pass 加载与执行的流程
=========================

2.1 clang: 从源码到 IR
------------------

### 2.1.1 main

`clang`的入口位于`clang/tools/driver/driver.cpp`中的`main`函数。

```
int main(int Argc, const char **Argv) {
    ...
    // 从第二个参数开始搜索第一个非空参数（跳过了参数命令行中的clang）
    auto FirstArg = llvm::find_if(llvm::drop_begin(Args), [](const char *A) { return A != nullptr; });
    //如果FirstArg以"-cc1"开头
    if (FirstArg != Args.end() && StringRef(*FirstArg).startswith("-cc1")) {
        // If -cc1 came from a response file, remove the EOL sentinels.
        if (MarkEOLs) {
            auto newEnd = std::remove(Args.begin(), Args.end(), nullptr);
            Args.resize(newEnd - Args.begin());
        }
        return ExecuteCC1Tool(Args);//调用ExecuteCC1Tool函数进一步处理
    }
    // 在Driver::BuildCompilation()中真正的命令行解析之前，处理需要处理的选项
    ...
    //初始化driver
    Driver TheDriver(Path, llvm::sys::getDefaultTargetTriple(), Diags);
    ...
    //如果不需要在新的进程中调用cc1工具
    if (!UseNewCC1Process) {
        TheDriver.CC1Main = &ExecuteCC1Tool;//CC1Main指向ExecuteCC1Tool函数
        llvm::CrashRecoveryContext::Enable();
    }
    //调用BuildCompilation构建编译任务，里面会将-cc1加入到Args中
    std::unique_ptr C(TheDriver.BuildCompilation(Args));
    int Res = 1;
    bool IsCrash = false;
    if (C && !C->containsError()) {
        SmallVector, 4> FailingCommands;
        //执行编译任务,里面会创建多进程，回到main函数开始的地方，执行ExecuteCC1Tool函数
        Res = TheDriver.ExecuteCompilation(*C, FailingCommands);
        // Force a crash to test the diagnostics.
        ...
        //处理执行编译命令时的失败情况
        ...
    }
    ...
} 
```

其中第 25 行的`BuildCompilation`函数以及第 31 行的`ExecuteCompilation`函数，它们的进一步跟进请参考[谁说不能与龙一起跳舞：Clang / LLVM (3) - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/23040952)。简单来说，一开始的命令行参数并不会满足代码中第 6 行的要求（即没有`-cc1`），从而一开始不会执行`ExecuteCC1Tool`函数，但通过一系列操作，最终还是执行了`ExecuteCC1Tool`函数。

### 2.1.2 ExecuteCC1Tool

`ExecuteCC1Tool`的具体实现在`clang/tools/driver/driver.cpp`中。

```
static int ExecuteCC1Tool(SmallVectorImpl &ArgV) {
    ...
    StringRef Tool = ArgV[1];
    void *GetExecutablePathVP = (void *)(intptr_t)GetExecutablePath;
    if (Tool == "-cc1")
        return cc1_main(makeArrayRef(ArgV).slice(1), ArgV[0], GetExecutablePathVP);
    if (Tool == "-cc1as")
        return cc1as_main(makeArrayRef(ArgV).slice(2), ArgV[0],
                          GetExecutablePathVP);
    if (Tool == "-cc1gen-reproducer")
        return cc1gen_reproducer_main(makeArrayRef(ArgV).slice(2), ArgV[0],
                                      GetExecutablePathVP);
    ...
} 
```

在上一小节中，我们知道了第一个参数是`-cc1`，因此这里会调用`cc1_main`函数。

### 2.1.3 cc1_main

`cc1_main`的具体实现在`clang/tools/driver/cc1_main.cpp`中。

```
int cc1_main(ArrayRef Argv, const char *Argv0, void *MainAddr) {
    ...
    //创建Clang编译器实例
    std::unique_ptr Clang(new CompilerInstance());
    IntrusiveRefCntPtr DiagID(new DiagnosticIDs());
    //注册了支持对象文件封装的 Clang 模块
    ...
    // 一系列初始化
    ...
    // 执行clang的前端
    {
        llvm::TimeTraceScope TimeScope("ExecuteCompiler");
        //调用ExecuteCompilerInvocation函数
        Success = ExecuteCompilerInvocation(Clang.get());
    }
    //后续的清理工作
    ...
} 
```

这里主要是创建`clang`实例，调用`ExecuteCompilerInvocation`函数开始编译目标源代码。

### 2.1.4 clang::ExecuteCompilerInvocation

`ExecuteCompilerInvocation`函数的声明在`clang/include/clang/FrontendTool/ExecuteCompilerInvocation.h`中，具体实现在`clang/lib/FrontendTool/ExecuteCompilerInvocation.cpp`中。

```
bool ExecuteCompilerInvocation(CompilerInstance *Clang) {
    // 命令行参数中是否存在-help、-v的参数，是则返回对应的信息
    ...
    // 加载必要插件
    Clang->LoadRequestedPlugins();
 
    // 同样还是检查一些参数，例如-mllvm
    ...
    // 调用CreateFrontendAction函数创建前端操作对象
    std::unique_ptr Act(CreateFrontendAction(*Clang));
    if (!Act)
        return false;
       // 调用ExecuteAction函数，执行前端操作,也就是编译
    bool Success = Clang->ExecuteAction(*Act);
    ...
    return Success;
} 
```

这里主要是创建`FrontendAction`对象并执行`ExecuteAction`函数。

### 2.1.5 clang::CompilerInstance::ExecuteAction

`CompilerInstance`类的声明在`clang/include/clang/Frontend/CompilerInstance.h`中，具体实现在`clang/lib/Frontend/CompilerInstance.cpp`中。

```
bool CompilerInstance::ExecuteAction(FrontendAction &Act) {
       //准备工作和选项处理
    ...
    for (const FrontendInputFile &FIF : getFrontendOpts().Inputs) {
        if (hasSourceManager() && !Act.isModelParsingAction())
            getSourceManager().clearIDTables();
        //BeginSourceFile开始处理源文件
        if (Act.BeginSourceFile(*this, FIF)) {
            //调用Execute函数进行处理
            if (llvm::Error Err = Act.Execute()) {
                consumeError(std::move(Err));
            }
            //结束
            Act.EndSourceFile();
        }
    }
    //错误处理
    ...
    //生成代码输出
    ...
    return !getDiagnostics().getClient()->getNumErrors();
}

```

这里通过`BeginSourceFile`函数加载源文件到内存中了，然后调用了`FrontendAction`类的`Execute`函数进行编译。

### 2.1.6 clang::FrontendAction::Execute

`FrontendAction`类的声明在`clang/include/clang/Frontend/FrontendAction.h`中，具体实现在`clang/lib/Frontend/FrontendAction.cpp`中。

```
llvm::Error FrontendAction::Execute() {
    //获取编译器实例的引用
    CompilerInstance &CI = getCompilerInstance();
 
    if (CI.hasFrontendTimer()) {
        ...
        ExecuteAction();//
    }
    else ExecuteAction();
 
    // If we are supposed to rebuild the global module index, do so now unless
    // there were any module-build failures.
    ...
     
    return llvm::Error::success();
}

```

该函数进一步调用`ASTFrontendAction`类的`ExecuteAction`函数。

### 2.1.7 clang::ASTFrontendAction::ExecuteAction

这个函数在开头的函数调用堆栈图中并没有出现，然而实际上确实调用了（真不明白这个是什么原因），如下图所示：

![](https://bbs.kanxue.com/upload/attach/202406/985561_BQDHUTTGYAPTK7F.webp)

那么就来看一下这个函数的源码。`ASTFrontendAction`类的声明在`clang/include/clang/Frontend/FrontendAction.h`中，具体实现在`clang/lib/Frontend/FrontendAction.cpp`中。

```
void ASTFrontendAction::ExecuteAction() {
    ...
    //没有语义分析器则创建
    if (!CI.hasSema())
        CI.createSema(getTranslationUnitKind(), CompletionConsumer);
    //调用ParseAST分析AST语法树
    ParseAST(CI.getSema(), CI.getFrontendOpts().ShowStats, CI.getFrontendOpts().SkipFunctionBodies);
}

```

主要是创建语义分析器，调用 `ParseAST` 方法，开始解析抽象语法树（AST）。

### 2.1.8 clang::ParseAST

`ParseAST`函数的声明在`clang/include/clang/Parse/ParseAST.h`中，具体实现在`clang/lib/Parse/ParseAST.cpp`中。

```
void clang::ParseAST(Sema &S, bool PrintStats, bool SkipFunctionBodies) {
    ...
    //获取AST消费者
    ASTConsumer *Consumer = &S.getASTConsumer();
    //创建解析器
    std::unique_ptr ParseOP(new Parser(S.getPreprocessor(), S, SkipFunctionBodies));
    Parser &P = *ParseOP.get();
    ...
    //设置主源文件，开始处理主源文件中的内容
    S.getPreprocessor().EnterMainSourceFile();
    //获取外部AST源，外部AST源通常用于提供额外的语义信息或进行增量编译
    ExternalASTSource *External = S.getASTContext().getExternalSource();
    if (External)
        //通知外部源开始翻译单元的处理
        External->StartTranslationUnit(Consumer);
    //获取词法分析器
    bool HaveLexer = S.getPreprocessor().getCurrentLexer();
    if (HaveLexer) {
        llvm::TimeTraceScope TimeScope("Frontend");
        P.Initialize();
        Parser::DeclGroupPtrTy ADecl;//用于存储解析器解析的顶层声明组
        Sema::ModuleImportState ImportState;//模块导入的状态
        //PotentiallyEvaluated用于在语义分析期间设置表达式求值的上下文
        EnterExpressionEvaluationContext PotentiallyEvaluated(S, Sema::ExpressionEvaluationContext::PotentiallyEvaluated);
        //解析源文件中的顶层声明，并将它们传递给 AST 消费者进行处理
        for (bool AtEOF = P.ParseFirstTopLevelDecl(ADecl, ImportState); !AtEOF; AtEOF = P.ParseTopLevelDecl(ADecl, ImportState)) {
            if (ADecl && !Consumer->HandleTopLevelDecl(ADecl.get()))
                return;
        }
    }
 
    // 处理由#pragma weak生成的顶层声明
    for (Decl *D : S.WeakTopLevelDecls())
        Consumer->HandleTopLevelDecl(DeclGroupRef(D));
    Consumer->HandleTranslationUnit(S.getASTContext());
 
    //收尾工作
    ...
} 
```

这部分真正是对源码进行语法树构建，并通过`HandleTranslationUnit`函数交给`AST Consumer`处理。

### 2.1.9 clang::BackendConsumer::HandleTranslationUnit

`BackendConsumer`类的声明在`clang/include/clang/CodeGen/CodeGenAction.h`中，具体实现在`clang/lib/CodeGen/CodeGenAction.cpp`中。

```
void BackendConsumer::HandleTranslationUnit(ASTContext &C) {
    {
        ...
        //调用了一个叫做 HandleTranslationUnit 的函数来处理翻译单元
        Gen->HandleTranslationUnit(C);
        ...
    }
    ...
    //设置 LLVM 的诊断处理程序和配置优化记录文件
    ...
    // 链接LinkModule到我们得模块中
    if (LinkInModules(getModule()))
        return;
    //遍历模块中的函数
    for (auto &F : getModule()->functions()) {
        //通过函数名获取对应的声明
        if (const Decl *FD = Gen->GetDeclForMangledName(F.getName())) {
            //将函数名的哈希值和声明的位置信息存储在ManglingFullSourceLocs中
            auto Loc = FD->getASTContext().getFullLoc(FD->getLocation());
            auto NameHash = llvm::hash_value(F.getName());
            ManglingFullSourceLocs.push_back(std::make_pair(NameHash, Loc));
        }
    }
    ...
    //EmbedBitcode用于处理-fembed-bitcode参数，目的是用于在生成的obj文件中增加一个用于存放bitcode的section
    EmbedBitcode(getModule(), CodeGenOpts, llvm::MemoryBufferRef());
    //clang后端代码生成（.ll或.bc文件）
    EmitBackendOutput(Diags, HeaderSearchOpts, CodeGenOpts, TargetOpts, LangOpts,
                      C.getTargetInfo().getDataLayoutString(), getModule(),
                      Action, FS, std::move(AsmOutStream), this);
    ...
}

```

该函数主要是记录了模块中函数的函数名哈希值和声明位置信息，最后调用`EmitBackendOutput`函数生成中间代码。

> **什么是模块？**
> 
> 在 LLVM 中，"模块（Module）" 通常是指一个编译单元或一个源代码文件被编译后生成的中间表示（IR，Intermediate Representation）的集合。在 LLVM 中，每个模块都是一个独立的单元，包含了函数、全局变量、类型定义等信息，可以被独立地优化和编译。

### 2.1.10 clang::EmitBackendOutput

`EmitBackendOutput`函数声明在`clang/include/clang/CodeGen/BackendUtil.h`，具体实现在`clang/lib/CodeGen/BackendUtil.cpp`中。

```
void clang::EmitBackendOutput(DiagnosticsEngine &Diags,
                              const HeaderSearchOptions &HeaderOpts,
                              const CodeGenOptions &CGOpts,
                              const clang::TargetOptions &TOpts,
                              const LangOptions &LOpts,
                              StringRef TDesc, Module *M,
                              BackendAction Action,
                              std::unique_ptr OS) {
    ...
    //检查是否启用了ThinLTO(链接时优化技术)，并进行相应操作
    ...
    //创建了EmitAssemblyHelper实例，用于辅助执行汇编输出的相关操作
    EmitAssemblyHelper AsmHelper(Diags, HeaderOpts, CGOpts, TOpts, LOpts, M);
    //根据Action执行相应的汇编操作
    if (CGOpts.LegacyPassManager)//旧版本使用LegacyPassManager
        AsmHelper.EmitAssemblyWithLegacyPassManager(Action, std::move(OS));
    else//新版本
        AsmHelper.EmitAssembly(Action, std::move(OS));
    // 验证生成的目标代码是否与目标描述一致
    if (AsmHelper.TM) {
        std::string DLDesc = M->getDataLayout().getStringRepresentation();
        ...
    }
} 
```

最终通过`BackendConsumer`将 AST 转换成了 IR 代码，之后`CGOpts.LegacyPassManager`标志选择执行新版本的`EmitAssemblyWithLegacyPassManager`或是旧版本的`EmitAssembly`。

> Clang 的后端消费者（BackendConsumer）是 Clang 的一部分，它负责将 Clang 前端产生的抽象语法树（AST）转换为 LLVM 的中间表示（IR）

2.2 Pass 加载
-----------

很奇怪，这一部分的`EmitAssemblyHelper`类的函数无法触发断点，且提示没有加载进来该函数。

### 2.2.1 EmitAssemblyHelper::EmitAssemblyWithLegacyPassManager

该函数同样也在`clang/lib/CodeGen/BackendUtil.cpp`中。

```
void EmitAssemblyHelper::EmitAssemblyWithLegacyPassManager(
    ...
    DebugifyCustomPassManager PerModulePasses;//创建模块Pass管理器
    ...
    PerModulePasses.add(createTargetTransformInfoWrapperPass(getTargetIRAnalysis()));//将目标IR分析包装成Pass并加入到模块Pass管理器中
    legacy::FunctionPassManager PerFunctionPasses(TheModule);//创建函数Pass管理器
    PerFunctionPasses.add(createTargetTransformInfoWrapperPass(getTargetIRAnalysis()));
    //创建Pass，并添加到对应的Pass管理器的执行队列中
    CreatePasses(PerModulePasses, PerFunctionPasses);
    ...
    legacy::PassManager CodeGenPasses;
    CodeGenPasses.add(createTargetTransformInfoWrapperPass(getTargetIRAnalysis()));
    //根据Action执行相应操作
    ...
    {...
        //对模块中已声明的函数进行Pass优化
        PerFunctionPasses.doInitialization();
        for (Function &F : *TheModule)
            if (!F.isDeclaration())
                PerFunctionPasses.run(F);
        PerFunctionPasses.doFinalization();
    }
    {...
        PerModulePasses.run(*TheModule);//对每个模块进行Pass优化
    }
    {...
        CodeGenPasses.run(*TheModule);
    }
    //保留输出文件
    ...
}

```

这部分代码主要是创建出两个重要的`Pass`管理器：`PerModulePasses`、`PerFunctionPasses`。然后调用`CreatePasses`函数创建`Pass`并添加到对应的`Pass`管理器的执行队列中（详见 2.2.2 小节）。之后就是调用`PerFunctionPasses.run(F)`、`PerModulePasses.run(*TheModule)`、`CodeGenPasses.run(*TheModule)`来执行`Pass`（详见 2.3 小节，以`PerModulePasses.run`函数为例进行讲解）。

### 2.2.2 EmitAssemblyHelper::CreatePasses

`CreatePasses`函数同样也在`clang/lib/CodeGen/BackendUtil.cpp`中。

```
void EmitAssemblyHelper::CreatePasses(legacy::PassManager &MPM, legacy::FunctionPassManager &FPM) {
    ...
    //创建PassManagerBuilderWrapper，用于配置和设置 Pass
    PassManagerBuilderWrapper PMBuilder(TargetTriple, CodeGenOpts, LangOpts);
    // 根据优化等级执行操作
    ...
    //设置PMBuilder的一些参数
    ...
    // 添加针对特定需求的扩展Pass, 通过Extensions向量来存储，此时并没有进行执行队列中
    ...
    // Set up the per-function pass manager.
    ...
    // Set up the per-module pass manager.
    ...
    //将配置好的Pass添加到函数级别和模块级别的Pass管理器中，进入执行队列中
    PMBuilder.populateFunctionPassManager(FPM);
    PMBuilder.populateModulePassManager(MPM);
}

```

最后调用的`populateModulePassManager`、`populateFunctionPassManager`函数将`Pass`添加到对应管理器的执行队列中（详见 2.2.3 小节，以`populateModulePassManager`函数为例进行讲解）。

### 2.2.3 PassManagerBuilder::populateModulePassManager

`populateModulePassManager`函数在`llvm/lib/Transforms/IPO/PassManagerBuilder.cpp`中。这个函数想必大家都很熟悉，因为在之前的`OLLVM`移植、编写自己的`Pass`都需要在这里面进行添加。

```
void PassManagerBuilder::populateModulePassManager(legacy::PassManagerBase &MPM) {
    //一系列Pass的添加
    MPM.add(createAnnotation2MetadataLegacyPass());
    ...
}

```

到这里，可以认为我们的`Pass`已经创建好了，不再进行进一步深究。

2.3 Pass 执行
-----------

### 2.3.1 PassManager::run

`PerModulePasses.run`在`llvm/lib/IR/LegacyPassManager.cpp`中。

```
bool PassManager::run(Module &M) {
  return PM->run(M);
}

```

最终调用`PassManagerImpl::run`函数

### 2.3.2 llvm::legacy::PassManagerImpl

`PassManagerImpl`类的定义在`llvm/include/llvm/IR/LegacyPassManager.h`中，具体实现在`llvm/lib/IR/LegacyPassManager.cpp`中。

```
bool PassManagerImpl::run(Module &M) {
    ...
    //对所有的不可变 Pass 进行初始化
    for (ImmutablePass *ImPass : getImmutablePasses())
        Changed |= ImPass->doInitialization(M);//执行 Pass 的初始化工作
    initializeAllAnalysisInfo();//初始化所有的分析信息
    //遍历了所有包含在 Pass 管理器中的子管理器（Manager）
    for (unsigned Index = 0; Index < getNumContainedManagers(); ++Index) {
        //对模块M执行Pass
        Changed |= getContainedManager(Index)->runOnModule(M);
        M.getContext().yield();
    }
    //对所有的不可变 Pass 进行收尾工作
    for (ImmutablePass *ImPass : getImmutablePasses())
        Changed |= ImPass->doFinalization(M);
 
    return Changed;
}

```

该函数主要是对模块执行所有被安排执行的`Pass`，对应代码中第 15~19 行，关键函数为`runOnModule`。

> **什么是不可变 Pass？**  
> 在 LLVM 中，Pass（通常称为优化 Pass 或者分析 Pass）是指一种对 LLVM IR 进行转换或者分析的模块。Passes 可以用于执行各种任务，例如优化代码、收集统计信息、生成调试信息等。Passes 通常根据其行为被分为两类：可变 Pass 和不可变 Pass。
> 
> 1.  **可变 Pass（Mutable Pass）**：可变 Pass 是指在执行过程中可以修改 LLVM IR 的 Pass。这意味着它们可以插入、删除或修改指令、函数、基本块等内容。可变 Pass 通常用于优化编译过程，例如执行指令调度、函数内联等。
> 2.  **不可变 Pass（Immutable Pass）**：不可变 Pass 是指在执行过程中不会修改 LLVM IR 的 Pass。它们只会读取 IR 并执行一些分析或者只读的转换。不可变 Pass 通常用于收集信息、生成报告、进行静态分析等任务。

### 2.3.3 FPPassManager::runOnModule

`FPPassManager`类的定义在`llvm/include/llvm/IR/LegacyPassManagers.h`中，具体实现在`llvm/lib/IR/LegacyPassManager.cpp`中。

```
bool FPPassManager::runOnModule(Module &M) {
  bool Changed = false;
  
  for (Function &F : M)
    Changed |= runOnFunction(F);
  
  return Changed;
}

```

调用`runOnFunction`函数对模块中的函数进行 Pass 操作。

### 2.3.4 FPPassManager::runOnFunction

`runOnFunction`函数具体实现在`llvm/lib/IR/LegacyPassManager.cpp`中。

```
bool FPPassManager::runOnFunction(Function &F) {
    //首先检查函数是否是一个声明（declaration），如果是的话，说明这个函数只是声明但没有定义，那么就没有必要对其进行优化
    if (F.isDeclaration()) return false;
    ...
    Module &M = *F.getParent();//获取函数所在模块
    //从模块级别的pass manager中获取继承的分析（analysis）信息
    populateInheritedAnalysis(TPM->activeStack);
 
    // Collect the initial size of the module.
    ...
 
    // Store name outside of loop to avoid redundant calls.
    const StringRef Name = F.getName();
    llvm::TimeTraceScope FunctionScope("OptFunction", Name);
    //遍历需要执行的pass列表
    for (unsigned Index = 0; Index < getNumContainedPasses(); ++Index) {
        FunctionPass *FP = getContainedPass(Index);//获取当前pass
        bool LocalChanged = false;
        ...
        //初始化分析 Pass
        initializeAnalysisImpl(FP);
        {
            PassManagerPrettyStackEntry X(FP, F);
            TimeRegion PassTimer(getPassTimer(FP));
            #ifdef EXPENSIVE_CHECKS
            uint64_t RefHash = FP->structuralHash(F);
            #endif
            LocalChanged |= FP->runOnFunction(F);//执行Pass
            ...
        }
        //清理工作
        ...
    }
    return Changed;
}

```

最后通过`runOnFunction`执行对应的`Pass`（代码中第 28 行），在当前例子中，也就是执行`MyPass`的`runOnFunction`函数。

三、结语
====

以上就是。简而言之，首先`clang`会先将我们的目标源码转成 AST 语法树，然后再通过`ASTConsumer`换成`IR`代码。之后加载通过`CreatePasses`函数创建 Pass 并加入到执行队列中，后续会调用我们熟知的`populateModulePassManager`，这里注册过我们自定义的`Pass`。最后就是`Pass`执行，主要还是通过对应的`Pass`管理器的`runOnModule`函数来进行的，它这里面会直接调用我们自定义`Pass`的`runOnModule`函数。值得一提的是，`Pass`的加载与执行的操作都是在`EmitAssemblyWithLegacyPassManager`或`EmitAssembly`函数中调用和完成的。

参考：

[【Linux】GDB 保姆级调试指南（什么是 GDB？GDB 如何使用？）_linux gdb 标准输入 - CSDN 博客](https://blog.csdn.net/weixin_45031801/article/details/134399664)

[对 LLVM Pass 进行 Debug_vscode llvm pass 开发 - CSDN 博客](https://blog.csdn.net/qq_39209008/article/details/126560080)

[谁说不能与龙一起跳舞：Clang / LLVM (3) - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/23040952)

[llvm 学习（二十）：动态注册 Pass 的加载过程（上） | LeadroyaL's website](https://leadroyal.cn/p/2207/)

[(3 封私信 / 1 条消息) Clang 里面真正的前端是什么？ - 知乎 (zhihu.com)](https://www.zhihu.com/question/31425289)

[https://juejin.cn/post/6844903591115767821](https://juejin.cn/post/6844903591115767821)

[[培训] 内核驱动高级班，冲击 BAT 一流互联网大厂工作，每周日 13:00-18:00 直播授课](https://www.kanxue.com/book-section_list-174.htm)

[#混淆加固](forum-161-1-121.htm) [#源码框架](forum-161-1-127.htm)