> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [polarply.medium.com](https://polarply.medium.com/build-your-first-llvm-obfuscator-80d16583392b)

> Welcome to a tutorial on building your first LLVM based obfuscator. In this post we will list the adv......

Welcome to a tutorial on building your first LLVM based obfuscator!  
In this post we will briefly present LLVM, discuss popular obfuscation approaches and their shortcomings and build our own epic LLVM-based string obfuscator.

All the code I present in this article is also available in [Github](https://github.com/tsarpaul/llvm-string-obfuscator).

**LLVM**

LLVM is a compiler framework built with the purpose of reducing time and cost of constructing new language compilers. With LLVM, all a language has to do is implement a “front-end” to LLVM. This means it needs to transform its code into LLVM Intermediate Representation (IR) code and LLVM will take care of all the ugly architecture/platform-specific machine code generation:

![](https://miro.medium.com/max/1276/0*ruIvRn0Eldfontf_)

Popular compilers that are really just LLVM front-ends are Clang, rustc (Rust’s compiler) and Swift’s compiler.

LLVM offers a system for inserting optimization plugins which review Intermediate Representation and make changes to it.

This system offers exactly what we need for our purposes. We will register our obfuscator in the optimization pipeline illustrated above and have access to LLVM IR code.

LLVM’s biggest competitor, GCC also has a plugin system however it seemed less attractive. Modularity is a keystone in LLVM’s architecture, which makes it very convenient for us.

**Why obfuscate through bytecode?**

For the most part open-source obfuscation projects use macro-based techniques, eg:

```
void SampleFiniteStateMachine()
{
  using namespace andrivet::ADVobfuscator::Machine1;  OBFUSCATED_CALL0(FunctionToProtect);
  auto result = OBFUSCATED_CALL_RET(int, FunctionToProtectWithParameters, OBFUSCATED(“did”), OBFUSCATED(“again”));
  cout << “Result: “ << result << endl;
}
```

[https://github.com/andrivet/ADVobfuscator](https://github.com/andrivet/ADVobfuscator)

Where OBFUSCATED_CALL0, OBFUSCATED are macros designed to inject the code obfuscation transformations for the former and string obfuscations for the latter.

While these projects enable authors to control exactly where their obfuscation schemes will apply, they taint the source-code and for larger projects they can become annoying to manage.

Another big drawback is that such tools won’t work with other languages since they use language-specific features.

Another less popular technique manipulates source-code directly before compiling, and while retaining the original source-code’s elegance it still lacks cross-language support.

Our obfuscator will work on a lower level — the common-ground for many languages, the LLVM bytecode. This will enable us to keep our source code clean while supporting multiple languages, without even being aware of the language we’ll be operating on.

**The Obfuscator**

To demonstrate LLVM’s power, we will implement a string obfuscator, which will encode every string in the binary with ROR-1.

First we register our plugin with LLVM’s optimizations pipeline:

```
extern “C” ::llvm::PassPluginLibraryInfo LLVM_ATTRIBUTE_WEAK
llvmGetPassPluginInfo() {
  return {
    LLVM_PLUGIN_API_VERSION, “StringObfuscatorPass”, “v0.1”, [](PassBuilder &PB) {
      PB.registerPipelineParsingCallback([](StringRef Name, ModulePassManager &MPM,
      ArrayRef<PassBuilder::PipelineElement>) {
        if(Name == “string-obfuscator-pass”){
          MPM.addPass(StringObfuscatorModPass());
          return true;
        }
        return false;
        }
      );
    }
  };
}
```

LLVM allows registering optimizations (also known as passes) at different stages. You may register your pass to run on every module, function, block or even loop.

In LLVM, a module is a set of code that is processed together, e.g. a .cpp file. This is where we’ll register our plugin.

The plugin will activate when the “string-obfuscator-pass” keyword is passed to the optimizer.

We then define the plugin itself, which inherits the PassInfoMixin class:

```
struct StringObfuscatorModPass : public PassInfoMixin<StringObfuscatorModPass> {
  PreservedAnalyses run(Module &M, ModuleAnalysisManager &MAM) {    // Transform the strings
    auto GlobalStrings = encodeGlobalStrings(M);    // Inject functions
    Function *DecodeFunc = createDecodeFunc(M);
    Function *DecodeStub = createDecodeStubFunc(M, GlobalStrings, DecodeFunc);    // Inject a call to DecodeStub from main
    Function *MainFunc = M.getFunction(“main”);
    createDecodeStubBlock(MainFunc, DecodeStub); return PreservedAnalyses::all();
  };
};
```

We only need to implement the _run_ function which is called on every module.

Formalities aside, we can now dive right into the action.

We immediately iterate all the global strings and encode them. It’s important to note that all static strings are defined as globals, regardless of their scope (even if they’re confined to just one function).

```
vector<GlobalString*> encodeGlobalStrings(Module& M){
  vector<GlobalString*> GlobalStrings;
  for(GlobalVariable &Glob : M.globals()) {
    // Ignore external globals & uninitialized globals.
    if (!Glob.hasInitializer() || Glob.hasExternalLinkage())
      continue;  // Unwrap the global variable to receive its value
  Constant *Initializer = Glob.getInitializer();

    // Check if its a string
  if (isa<ConstantDataArray>(Initializer)) {
    auto CDA = cast<ConstantDataArray>(Initializer);
    if (!CDA->isString())
      continue;

        // Extract raw string
    StringRef StrVal = CDA->getAsString();
    const char *Data = StrVal.begin();
    const int Size = StrVal.size();

    // Create encoded string variable (ROR1)
    char *NewData = EncodeString(Data, Size);
    Constant *NewConst = ConstantDataArray::getString(Ctx, StringRef(NewData, Size), false);    // Overwrite the global value
    Glob.setInitializer(NewConst);
    GlobalStrings.push_back(new GlobalString(&Glob));
    Glob.setConstant(false);
  }
…
  return GlobalStrings;
}
```

Globals are implemented through the GlobalVariable class in LLVM.

The GlobalVariable class doesn’t actually hold the value of the global itself. You might have uninitialized globals because they point to an external definition, so the value itself is decoupled from the GlobalVariable class and is represented by the Constant class. We grab the value, make sure it’s a string and apply ROR1.

Now we need to start injecting logic into the binary to decode our strings during runtime.

To do this we need to implement a decode function and pass those strings we encoded to this function.

To this end, we’ll need to inject our own LLVM bytecode, but I’m no LLVM assembly expert so we’ll cheat.

First we build our decode function in C:

```
void Decode(char *String){
  while(*String != ‘\x00’){
    *String -= 1;
     String++;
  }
}
```

We then compile this to LLVM IR through Clang:

```
$ clang -Os -shared -emit-llvm -c decode.c -o decode.bs && llvm-dis < decode.bs
```

decode.bs IR disassembly:

```
define dso_local void @Decode(i8* nocapture %String) local_unnamed_addr #0 {
entry:
%0 = load i8, i8* %String, align 1, !tbaa !2
  %cmp6 = icmp eq i8 %0, 0
  br i1 %cmp6, label %while.end, label %while.bodywhile.body: ; preds = %entry, %while.body
  %1 = phi i8 [ %2, %while.body ], [ %0, %entry ]
  %String.addr.07 = phi i8* [ %incdec.ptr, %while.body ], [ %String, %entry ]
  %sub = add i8 %1, -1
  store i8 %sub, i8* %String.addr.07, align 1, !tbaa !2
  %incdec.ptr = getelementptr inbounds i8, i8* %String.addr.07, i64 1
  %2 = load i8, i8* %incdec.ptr, align 1, !tbaa !2
  %cmp = icmp eq i8 %2, 0
  br i1 %cmp, label %while.end, label %while.bodywhile.end: ; preds = %while.body, %entry
  ret void
}
```

Using this IR dump we build our function manually:

```
Function *createDecodeFunc(Module &M){
auto &Ctx = M.getContext();// Add Decode function
FunctionCallee DecodeFuncCallee = M.getOrInsertFunction(“decode”, /*ret*/ Type::getVoidTy(Ctx), /*args*/ Type::getInt8PtrTy(Ctx, 8));
Function *DecodeFunc = cast<Function>(DecodeFuncCallee.getCallee());
DecodeFunc->setCallingConv(CallingConv::C);// Name DecodeFunc arguments
Function::arg_iterator Args = DecodeFunc->arg_begin();
Value *StrPtr = Args++;
StrPtr->setName(“StrPtr”);// Create blocks
BasicBlock *BEntry = BasicBlock::Create(Ctx, “entry”, DecodeFunc);
BasicBlock *BWhileBody = BasicBlock::Create(Ctx, “while.body”, DecodeFunc);
BasicBlock *BWhileEnd = BasicBlock::Create(Ctx, “while.end”, DecodeFunc);// Entry block
IRBuilder<> *Builder = new IRBuilder<>(BEntry);
auto *var0 = Builder->CreateLoad(StrPtr, “var0”);
auto *cmp6 = Builder->CreateICmpEQ(var0, Constant::getNullValue(Type::getInt8Ty(Ctx)), “cmp6”);
Builder->CreateCondBr(cmp6, BWhileEnd, BWhileBody);// Preheader block
Builder = new IRBuilder<>(BWhileBody);
PHINode *var1 = Builder->CreatePHI(Type::getInt8Ty(Ctx), 2, “var1”);
PHINode *stringaddr07 = Builder->CreatePHI(Type::getInt8PtrTy(Ctx, 8), 2, “stringaddr07”);
auto *sub = Builder->CreateSub(var1, ConstantInt::get(IntegerType::get(Ctx, 8), 1, true));
Builder->CreateStore(sub, stringaddr07);
auto *incdecptr = Builder->CreateGEP(stringaddr07, ConstantInt::get(IntegerType::get(Ctx, 64), 1), “incdecptr”);
auto *var2 = Builder->CreateLoad(incdecptr, “var2”);
auto cmp = Builder->CreateICmpEQ(var2, ConstantInt::get(IntegerType::get(Ctx, 8), 0), “cmp”);
Builder->CreateCondBr(cmp, BWhileEnd, BWhileBody);
…
```

EDIT: It was pointed out to me by [@0xrepnz](https://twitter.com/0xrepnz) that you could probably use the llvm::parseIRFile() API to load the IR bytecode for a more elegant solution, thanks 0xrepnz!

Lets run a quick test and see how our optimization affects the file. We can see our new function, however nobody’s using it yet:

![](https://miro.medium.com/max/576/0*SLoiSgZ6zm6ex1Bj)

Let’s put our new function in use then. We add another function which just calls the decode function on every string we transformed:

```
Function *createDecodeStubFunc(Module &M, vector<GlobalString*> &GlobalStrings, Function *DecodeFunc){
  Context &Ctx = M.getContext();

    // Add DecodeStub function
  FunctionCallee DecodeStubCallee = M.getOrInsertFunction(“decode_stub”, /*ret*/ Type::getVoidTy(Ctx));
  Function *DecodeStubFunc = cast<Function>(DecodeStubCallee.getCallee());
  DecodeStubFunc->setCallingConv(CallingConv::C);

  // Create entry block
  BasicBlock *BB = BasicBlock::Create(Ctx, “entry”, DecodeStubFunc);
  IRBuilder<> Builder(BB);  // Add calls to decode every encoded global
  for(GlobalString *GlobString : GlobalStrings){
    if(GlobString->type == GlobString->SIMPLE_STRING_TYPE){
    auto *StrPtr = Builder.CreatePointerCast(GlobString->Glob, Type::getInt8PtrTy(Ctx, 8));
    Builder.CreateCall(DecodeFunc, {StrPtr});
  }
…
  return DecodeStubFunc;
}
```

Beautiful:

![](https://miro.medium.com/max/822/0*ymb9btPDbIH66ZQU)

We’re almost done, lets just inject the call to the decode stub in _main_:

```
void createDecodeStubBlock(Function *F, Function *DecodeStubFunc){
  Context &Ctx = F->getContext();
  BasicBlock &EntryBlock = F->getEntryBlock();  // Create new block
  BasicBlock *NewBB = BasicBlock::Create(Ctx, “DecodeStub”, EntryBlock.getParent(), &EntryBlock);
  IRBuilder<> Builder(NewBB);  // Call stub func instruction
  Builder.CreateCall(DecodeStubFunc);  // Jump to original entry block
  Builder.CreateBr(&EntryBlock);
}run(…){
…
Function *MainFunc = M.getFunction(“main”);
createDecodeStubBlock(MainFunc, DecodeStub);
…
}
```

We can see that just a small call was added to the original main:

![](https://miro.medium.com/max/1400/0*-11Ws7SOttiXBY5b)

Finally, we return a status report to the LLVM Pass Manager:

```
PreservedAnalyses run(Module &M, ModuleAnalysisManager &MAM) {
…
  return PreservedAnalyses::all();
}
```

This simply means that we did not taint other passes’ results. This is important information for other passes, so they don’t need to be re-run.

I left instructions to compile the plugin in my repository. We can run the built plugin with LLVM’s optimizer **_opt_**:

```
# Create LLVM IR from C
$ clang -O3 -emit-llvm hello.c -c -o hello.bc# Run our plugin through the LLVM optimizer on the LLVM IR
$ opt -load-pass-plugin=/build/lib/LLVMStringObfuscator.so -passes=”string-obfuscator-pass” < hello.bc -o out.bc# LLVM Static Compiler
$ llc out.bc -o out.s# Link with clang
$ clang -static out.s -o out# Profit
$ ./out
```

We can view our output file’s disassembly to make sure our obfuscator works:

![](https://miro.medium.com/proxy/1*b2ilGXN_VwJ7tKczZKg5Sw.png)

**Summary**

This tutorial was an introduction to the potential of running obfuscations through LLVM. As LLVM becomes more mainstream this approach could become more popular.

You can take a look at a more serious approach for LLVM obfuscation at the link below, where the authors implemented more complex code obfuscation algorithms than our simple string obfuscation:  
https://github.com/obfuscator-llvm/obfuscator/

Thanks for reading!

**References**

[https://lwn.net/Articles/582697/](https://lwn.net/Articles/582697/) — LLVM vs GCC

[https://llvm.org/docs/LangRef.html](https://llvm.org/docs/LangRef.html) — LLVM reference

[https://medium.com/@mshockwave/writing-llvm-pass-in-2018-preface-6b90fa67ae82](https://medium.com/@mshockwave/writing-llvm-pass-in-2018-preface-6b90fa67ae82) — Modern LLVM passes