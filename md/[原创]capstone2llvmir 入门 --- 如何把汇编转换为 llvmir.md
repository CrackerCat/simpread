> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-267395.htm)

目录

*   [前言](#前言)
*   [capstone2llvmirtool 入门](#capstone2llvmirtool入门)
*            [对于不同代码的四种不同翻译方式](#对于不同代码的四种不同翻译方式)
*            [具体原理概括](#具体原理概括)
*                    首先，创建翻译模块 translator module
*                    然后，通过 translator 执行翻译
*   [capstone2llvmir 入口](#capstone2llvmir入口)
*   [translate 函数](#translate函数)
*   [translateInstruction 函数](#translateinstruction函数)
*   [translatePseudoAsmGeneric 函数](#translatepseudoasmgeneric函数)
*   [自己编译](#自己编译)
*   [retdectool](#retdectool)
*   [附录：capstone2llvmir 目录结构与源码](#附录：capstone2llvmir目录结构与源码)

前言
==

本文简单分析介绍了 capstone2llvmir 源码与本地编译运行的方式，适合初步学习汇编转 ir 的原理并自己做简单修改，编译运行，做出自己的简易 asm2llvmir 小程序，有了 llvmir，就可以优化、去混淆、干坏事了，详见本菜之前的文章

 

https://bbs.pediy.com/thread-265335.htm 利用编译器优化干掉虚假控制流

 

https://bbs.pediy.com/thread-266323.htm 利用编译器优化干掉控制流平坦化

 

ps：有些比较复杂的 asm2ir 转换源码里面没有，需要自己试着写，慢慢完善，然后编译成为自己的工具

 

recdec 的源代码里很重要的部分 capstone2llvmir 与 bin2llvmir，功能是把汇编转换为 llvmir，我认真学习了这个神器并记录笔记

 

源代码 https://retdec-tc.avast.com/repository/download/Retdec_DoxygenBuild/.lastSuccessful/build/doc/doxygen/html/files.html

 

它介绍里面有一个 [capstone2llvmirtool](https://github.com/avast/retdec/tree/master/src/capstone2llvmirtool) 入门 https://github.com/avast/retdec/wiki/Capstone2LlvmIr，我把它大概意思整理了一下

capstone2llvmirtool 入门
======================

对于不同代码的四种不同翻译方式
---------------

1. 完整语义翻译 完全把汇编语法翻译成 ir，只对于足够简单的指令 ps：很多不常用指令翻译源码里没有，如果碰到需要模仿源码自己写

 

2. 翻译为内部函数 call 把一些汇编翻译成大多数编译器理解的内部函数，比如翻译一些跳转

 

3. 翻译为伪代码 call 根据 Capstone 反汇编信息创建伪代码 call 翻译指令 ps：看到这些 call 对应的汇编没被翻译，而它对于优化又很重要，就可以着手自己写翻译函数了，不重要直接忽略就行

 

4. 不翻译 忽略一些难以翻译的指令

具体原理概括
------

### 首先，创建翻译模块 translator module

1. 创建空的 LLVM IR module

 

2. 初始化 Capstone engine 和其他数据结构

 

3. 创建架构运行环境，也就是寄存器相关数据结构什么的

 

3.1 把汇编地址映射为 ir 全局变量

 

@_asm_program_counter = internal global i64 0

 

; ...

 

; add eax, 0x1234 @ 0x1000

 

store volatile i64 4096, i64* @_asm_program_counter

 

; ... LLVM IR sequence for the add instruction

 

; sub ebx, 0x1234 @ 0x1005

 

store volatile i64 4101, i64* @_asm_program_counter

 

; ... LLVM IR sequence for the sub instruction

 

3.2 控制流伪代码函数生成，为什么不用 ir 是因为 ir 通过块标签跳转而不是像汇编一样通过地址

 

Control-flow-related pseudo functions are generated.

 

; void (i<architecture_size> target_address)

 

declare void @__pseudo_call(i32)

 

; void (i<architecture_size> target_address)

 

declare void @__pseudo_return(i32)

 

; void (i<architecture_size> target_address)

 

declare void @__pseudo_branch(i32)

 

; void (i1 condition, i<architecture_size> target_address)

 

declare void @__pseudo_cond_branch(i1, i32)

 

3.3 架构相关寄存器全局变量初始化

 

@eax = internal global i32 0

 

@ecx = internal global i32 0

 

; ...

 

@st0 = internal global x86_fp80 0xK00000000000000000000

 

@st1 = internal global x86_fp80 0xK00000000000000000000

### 然后，通过 translator 执行翻译

1. 用 Capstone engine 反编译二进制，对于一句汇编，它大概包含如下信息

 

`add eax, 0x1234`:

```
General info:
         id     :  8 (add)
         addr   :  1000
         size   :  5
         bytes  :  05 34 12 00 00
         mnem   :  add
         op str :  eax, 0x1234
 Detail info:
         R regs :  0
         W regs :  1
                 25 (eflags)
         groups :  0
 Architecture-dependent info:
         prefix :  00 00 00 00  (-, -, -, -)
         opcode :  05 00 00 00
         rex    :  0
         addr sz:  4
         modrm  :  0
         sib    :  0
         disp   :  0
         sib idx:  0 (-)
         sib sc :  0
         sib bs :  0 (-)
         sse cc :  X86_SSE_CC_INVALID
         avx cc :  X86_AVX_CC_INVALID
         avx sae:  false
         avx rm :  X86_AVX_RM_INVALID
         op cnt :  2
 
                 type   :  X86_OP_REG
                 reg    :  19 (eax)
                 size   :  4
                 access :  CS_AC_READ + CS_AC_WRITE
                 avx bct:  X86_AVX_BCAST_INVALID
                 avx 0 m:  false
 
                 type   :  X86_OP_IMM
                 imm    :  1234
                 size   :  4
                 access :  CS_AC_INVALID
                 avx bct:  X86_AVX_BCAST_INVALID
                 avx 0 m:  false

```

2. 找到翻译方式翻译指令到 ir id 保存了操作码

 

2.1Capstone ID is mapped to an ID-specific routine 每个 id 也就是操作码对应一个 routine

 

2.2Capstone ID is mapped to a specific pseudo assembly generation method

 

id 对应一个 pseudo method 汇编伪代码生成方法

```
__asm_(op0)
op0 = __asm_(op0)
__asm_(op0, op1)
op0 = __asm_(op1)
op0 = __asm_(op0, op1)
__asm_(op0, op1, op2) 
```

2.3Capstone ID is not mapped to any value

 

啥也没匹配到。使用 Capstone-provided instruction info 信息自动创建 call，这取决于 Capstone 提供信息的质量

 

源码结构：

 

公开接口 include/retdec/capstone2llvmir  
隐藏接口 src/capstone2llvmir

 

接口 Capstone2LlvmIrTranslator  
实现 Capstone2LlvmIrTranslator_impl  
相应架构实现 Capstone2LlvmIrTranslatorArm

capstone2llvmir 入口
==================

直接看入口，入口在 [capstone2llvmirtool](https://github.com/avast/retdec/tree/master/src/capstone2llvmirtool)/capstone2llvmir.cpp 里 main 函数 (还有一个在 retdec\src\bin2llvmir\optimizations\decoder 里，学习这 2 个函数，就能学会如何使用 translate 函数翻译 asm 为 ir)，先创建一个 llvm::function，填入一个 block 与 return，根据 cpu 架构创建翻译器 Capstone2LlvmIrTranslator::createArch，最后通过 capstone2llvmir/capstone2llvmir.h 定义的 translate 函数翻译 asm 为 ir，传入 data，size，base 获得 irb

```
main
{
 llvm::Function::Create
 llvm::BasicBlock::Create
 Capstone2LlvmIrTranslator::createArch
 translate(po.code.data(), po.code.size(), po.base, irb)
}

```

translate 函数
============

cs_malloc 分配 capstone 的 handle，用这个 handle 通过 cs_disasm_iter 把二进制翻译为汇编保存在 insn

 

generateSpecialAsm2LlvmInstr ，关键函数 generateSpecialAsm2LlvmInstr 把 insn 的 address 转换为 llvm 全局变量，每种架构都有一个程序计数器记录程序执行到哪个地址了，arm 就是 pc，每执行一句就修改 pc，这里的 pc 值就来源于 generateSpecialAsm2LlvmInstr 转换的 globalvalue

 

translateInstruction 真正进入到关键把 insn 翻译为 ir，这里 4 种方式对应前面的 4 种翻译策略，简单看一下骨架

```
Capstone2LlvmIrTranslator_impl::translate
{
 cs_malloc
 cs_disasm_iter 
 generateSpecialAsm2LlvmInstr
 translateInstruction //在capstone2llvmir_impl.h声明的虚函数，不同架构有不同的translateInstruction实现
 {
  *f=*(_i2fm.find(i→id)) //如果在Instruction translation map _i2fm里找到翻译函数，直接通过指针调用，对应1
  {
   translateAdd
   translateB
   ...
  }
  or translatePseudoAsmGeneric //如果没有找到，回到translatePseudoAsmGeneric函数，对应2
  {
   loadOp
   loadRegister
   getPseudoAsmFunction
   CreateCall                  //对应3
   storeOp
   storeRegister
  }
 }
} 
```

translateInstruction 函数
=======================

关键函数 translateInstruction，把汇编 insn 转换为 llvmir，它是 capstone2llvmir_impl.h 声明的一个虚函数

 

不同的汇编都有自己的 translateInstruction 实现，arm 的在 src\capstone2llvmir\arm\arm.cpp

 

这里面一个重要结构体`_cs_insn，电脑里的python3安装了capstone我们翻``python``看它的结构`

```
以前读cs的笔记：https://bbs.pediy.com/thread-258473.htm
class _cs_insn(ctypes.Structure):
    _fields_ = (
        ('id', ctypes.c_uint),
        ('address', ctypes.c_uint64),
        ('size', ctypes.c_uint16),
        ('bytes', ctypes.c_ubyte * 16),
        ('mnemonic', ctypes.c_char * 32),
        ('op_str', ctypes.c_char * 160),
        ('detail', ctypes.POINTER(_cs_detail)),
    )
class _cs_detail(ctypes.Structure):
    _fields_ = (
        ('regs_read', ctypes.c_uint16 * 12),
        ('regs_read_count', ctypes.c_ubyte),
        ('regs_write', ctypes.c_uint16 * 20),
        ('regs_write_count', ctypes.c_ubyte),
        ('groups', ctypes.c_ubyte * 8),
        ('groups_count', ctypes.c_ubyte),
        ('arch', _cs_arch),
    )
class _cs_arch(ctypes.Union):
    _fields_ = (
        ('arm64', arm64.CsArm64),
        ('arm', arm.CsArm),
        ('m68k', m68k.CsM68K),
        ('mips', mips.CsMips),
        ('x86', x86.CsX86),
        ('ppc', ppc.CsPpc),
        ('sparc', sparc.CsSparc),
        ('sysz', systemz.CsSysz),
        ('xcore', xcore.CsXcore),
        ('tms320c64x', tms320c64x.CsTMS320C64x),
        ('m680x', m680x.CsM680x),
        ('evm', evm.CsEvm),
    )   
/// Instruction structure
typedef struct cs_arm {
    bool usermode;    ///< User-mode registers to be loaded (for LDM/STM instructions)
    int vector_size;     ///< Scalar size for vector instructions
    arm_vectordata_type vector_data; ///< Data type for elements of vector instructions
    arm_cpsmode_type cps_mode;    ///< CPS mode for CPS instruction
    arm_cpsflag_type cps_flag;    ///< CPS mode for CPS instruction
    arm_cc cc;            ///< conditional code for this insn
    bool update_flags;    ///< does this insn update flags?
    bool writeback;        ///< does this insn write-back?
    arm_mem_barrier mem_barrier;    ///< Option for some memory barrier instructions
 
    /// Number of operands of this instruction,
    /// or 0 when instruction has no operand.
    uint8_t op_count;
 
    cs_arm_op operands[36];    ///< operands for this instruction.
} cs_arm; 
typedef enum arm_cc {
    ARM_CC_INVALID = 0,
    ARM_CC_EQ,            ///< Equal                      Equal
    ARM_CC_NE,            ///< Not equal                  Not equal, or unordered
    ARM_CC_HS,            ///< Carry set                  >, ==, or unordered
    ARM_CC_LO,            ///< Carry clear                Less than
    ARM_CC_MI,            ///< Minus, negative            Less than
    ARM_CC_PL,            ///< Plus, positive or zero     >, ==, or unordered
    ARM_CC_VS,            ///< Overflow                   Unordered
    ARM_CC_VC,            ///< No overflow                Not unordered
    ARM_CC_HI,            ///< Unsigned higher            Greater than, or unordered
    ARM_CC_LS,            ///< Unsigned lower or same     Less than or equal
    ARM_CC_GE,            ///< Greater than or equal      Greater than or equal
    ARM_CC_LT,            ///< Less than                  Less than, or unordered
    ARM_CC_GT,            ///< Greater than               Greater than
    ARM_CC_LE,            ///< Less than or equal         <, ==, or unordered
    ARM_CC_AL             ///< Always (unconditional)     Always (unconditional)
} arm_cc;
/// Instruction operand
typedef struct cs_arm_op {
    int vector_index;    ///< Vector Index for some vector operands (or -1 if irrelevant)
 
    struct {
        arm_shifter type;
        unsigned int value;
    } shift;
 
    arm_op_type type;    ///< operand type
 
    union {
        int reg;    ///< register value for REG/SYSREG operand
        int32_t imm;            ///< immediate value for C-IMM, P-IMM or IMM operand
        double fp;            ///< floating point value for FP operand
        arm_op_mem mem;        ///< base/index/scale/disp value for MEM operand
        arm_setend_type setend; ///< SETEND instruction's operand type
    };
 
    /// in some instructions, an operand can be subtracted or added to
    /// the base register,
    /// if TRUE, this operand is subtracted. otherwise, it is added.
    bool subtracted;
 
    /// How is this operand accessed? (READ, WRITE or READ|WRITE)
    /// This field is combined of cs_ac_type.
    /// NOTE: this field is irrelevant if engine is compiled in DIET mode.
    uint8_t access;
 
    /// Neon lane index for NEON instructions (or -1 if irrelevant)
    int8_t neon_lane;
} cs_arm_op;

```

translateInstruction 代码粗看

```
void Capstone2LlvmIrTranslatorArm_impl::translateInstruction(
        cs_insn* i,
        llvm::IRBuilder<>& irb)
{
    _insn = i;
 
    cs_detail* d = i->detail;
    cs_arm* ai = &d->arm;//这里储存了arm架构相关信息
 
    auto fIt = _i2fm.find(i->id);//这里id存储着指令类型
    //_i2fm是一个hash表在arm_init.cpp中初始化，存储部分arm指令与翻译方法一一对应，如ARM_INS_ADC对应Capstone2LlvmIrTranslatorArm_impl::translateAdc，可以看到还有很多指令还没有转换函数
    if (fIt != _i2fm.end() && fIt->second != nullptr)//如果在hash里找到了
    {
        auto f = fIt->second;//获得翻译方法f
 
        bool branchInsn = i->id == ARM_INS_B || i->id == ARM_INS_BX
                || i->id == ARM_INS_BL || i->id == ARM_INS_BLX
                || i->id == ARM_INS_CBZ || i->id == ARM_INS_CBNZ;
        if (ai->cc == ARM_CC_AL || ai->cc == ARM_CC_INVALID || branchInsn)
        //这里区分条件跳和非条件跳，cc就是condition code的意思
        {
            _inCondition = false;
            (this->*f)(i, ai, irb);//直接指针调用f处理irb
        }
        else
        {
            _inCondition = true;
 
            auto* cond = generateInsnConditionCode(irb, ai);//条件跳要generateIfThen先生成ifthen的bodyIrb
            auto bodyIrb = generateIfThen(cond, irb);
 
            (this->*f)(i, ai, bodyIrb);
        }
    }
    else
    {
        throwUnhandledInstructions(i);
 
        if (ai->cc == ARM_CC_AL || ai->cc == ARM_CC_INVALID)
        {
            _inCondition = false;
            translatePseudoAsmGeneric(i, ai, irb);//如果在_i2fm的hash表里没找到对应，继续用translatePseudoAsmGeneric生成ir，它定义在capstone2llvmir_impl.cpp里
        }
        else
        {
            _inCondition = true;
 
            auto* cond = generateInsnConditionCode(irb, ai);
            auto bodyIrb = generateIfThen(cond, irb);
 
            translatePseudoAsmGeneric(i, ai, bodyIrb);
        }
    }
}

```

这里面有一个重要的 hash 表_i2fm 全称 Instruction translation map，把汇编指令和翻译 ir 函数指针一一对应，比如 ARM_INS_ADC 加法指令对应指针 &Capstone2LlvmIrTranslatorArm_impl::translateAdc

 

还有 arm_init.cpp 中定义的寄存器符号名字对应的哈希表 r2n，寄存器符号类型对应的哈希表 r2t 这两个重要结构，他们完全抽象出了 arm 寄存器为 c++ 数据结构

translatePseudoAsmGeneric 函数
============================

翻译 asm 为一般的伪代码函数，就是处理在_i2fm 表里面没有对应翻译函数的指令如何翻译

 

1. 根据 capstone 提供的指令信息，搞明白要生成的 ir 有多少寄存器与非寄存器的读写，需要创建多少 llvm 的 type 和 value, 函数有没有返回值等信息

 

2. 根据之前创建的 llvm 的 type 和 value 创建参数和返回值，把__asm_ 与助记符 insn->mnemonic 拼接起来命名函数名字，生成一个空壳伪函数

 

3. 我们在生成 ir 的时候，如果观察到一些以汇编助记符命名的 ir 函数，就可以知道这句汇编指令没有对应的翻译函数，然后自己写一个完成完全的翻译，当然，_i2fm 表里面给的翻译函数 99％情况下够用了

```
void Capstone2LlvmIrTranslator_impl::translatePseudoAsmGeneric(
        cs_insn* i,
        CInsn* ci,
        llvm::IRBuilder<>& irb)//这里区分一下cs_insn是带有address，mnemonic，op_str，detail的信息很全的结构体，CInsn仅仅就是原始的汇编二进制指令结构
{
    std::vector vals;
    std::vector types;
 
    unsigned writeCnt = 0;
    llvm::Type* writeType = getDefaultType();
    bool writesOp = false;
    for (std::size_t j = 0; j < ci->op_count; ++j)//先遍历CInsn二进制汇编的operands读取寄存器相关信息，确定生成ir需要什么样的value和type
    {
        auto& op = ci->operands[j];
        auto access = getOperandAccess(op);//getOperandAccess获得operands是读取写入还是其他
// regs_read，字面理解是，返回存储所有读取的隐式寄存器的list，实测只有pc，lr，sp和状态寄存器会被存储在list中
// regs_write，字面理解是，返回存储所有写入的隐式寄存器的list，实测只有pc，lr，sp和状态寄存器会被存储在list中
// regs_access，合并上面2个的结果
// # Access types for instruction operands.
// CS_AC_INVALID  = 0        # Invalid/unitialized access type.
// CS_AC_READ     = (1 << 0) # Operand that is read from.
// CS_AC_WRITE    = (1 << 1) # Operand that is written to.
        if (access == CS_AC_INVALID || (access & CS_AC_READ))//如果有读取存在，调用loadOp翻译获得需要的llvm的value与type，存入vals与type向量
        {
            auto* o = loadOp(op, irb);
            vals.push_back(o);
            types.push_back(o->getType());
        }
 
        if (access & CS_AC_WRITE)//如果有写入寄存器，writesOp为真，调用getRegisterType获得寄存器类型llvm的value，存入vals向量
                                //如果不是写入寄存器，可能写入到内存地址之类的，直接默认存储到vals向量
        {
            writesOp = true;
            ++writeCnt;
 
            if (isOperandRegister(op))//如果写入寄存器
            {
                auto* t = getRegisterType(op.reg);
                if (writeCnt == 1 || writeType == t)
                {
                    writeType = t;
                }
                else
                {
                    writeType = getDefaultType();
                }
            }
            else
            {
                writeType = getDefaultType();
            }
        }
    }
 
    if (vals.empty())//如果上面遍历之后vals还是空，再次通过detail->regs_read_count遍历所有读取寄存器相关信息存入vals
    {
        // All registers must be ok, or don't use them at all.
        std::vector readRegs;
        readRegs.reserve(i->detail->regs_read_count);
        for (std::size_t j = 0; j < i->detail->regs_read_count; ++j)
        {
            auto r = i->detail->regs_read[j];
            if (getRegister(r))
            {
                readRegs.push_back(r);
            }
            else
            {
                readRegs.clear();
                break;
            }
        }
 
        for (auto r : readRegs)
        {
            auto* op = loadRegister(r, irb); //如果有读取寄存器操作，调用loadRegister获得irb
            vals.push_back(op);
            types.push_back(op->getType());
        }
    }
 
    auto* retType = writesOp ? writeType : irb.getVoidTy();//只要writesOp为真，retType就为返回类型，否则返回类型为void
    llvm::Function* fnc = getPseudoAsmFunction(//通过getPseudoAsmFunction创建翻译对应cs_insn* i，类型为types，返回值为retType的llvm函数原型
            i,                                 //注意这个函数只是一个空壳，是没有内部ir的，它通过getPseudoAsmFunctionName命名函数名字，就是把__asm_与助记符insn->mnemonic拼接起来
            retType,                           //这样等我们看到生成的ir时，就知道这句汇编指令没有对应的翻译函数，然后自己写一个类似Capstone2LlvmIrTranslatorArm_impl::translateAdc的翻译函数
            types);
 
    auto* c = irb.CreateCall(fnc, vals);//通过CreateCall创建参数为vals，原型为fnc的伪代码函数c
 
    std::set writtenRegs;
    if (retType)
    {
        for (std::size_t j = 0; j < ci->op_count; ++j)//先通过op_count遍历operands写入寄存器相关信息
        {
            auto& op = ci->operands[j];
            if (getOperandAccess(op) & CS_AC_WRITE)//Return (list-of-registers-read, list-of-registers-modified) by this instructions
            {
                storeOp(op, c, irb);//通过storeOp函数创建存储ir
 
                if (isOperandRegister(op))
                {
                    writtenRegs.insert(op.reg);//存储到被写入寄存器writtenRegs集合里
                }
            }
        }
    }
 
    // All registers must be ok, or don't use them at all.
    std::vector writeRegs;
    writeRegs.reserve(i->detail->regs_write_count);
    for (std::size_t j = 0; j < i->detail->regs_write_count; ++j)//再次通过detail遍历写入寄存器(不包含writtenRegs里被写入的寄存器)相关信息存储到writeRegs向量
    {
        auto r = i->detail->regs_write[j];
        if (writtenRegs.count(r))
        {
            // silently ignore
        }
        else if (getRegister(r))
        {
            writeRegs.push_back(r);
        }
        else
        {
            writeRegs.clear();
            break;
        }
    }
 
    for (auto r : writeRegs)
    {
        llvm::Value* val = retType->isVoidTy()
                ? llvm::cast(
                        llvm::UndefValue::get(getRegisterType(r)))
                : llvm::cast(c);
        storeRegister(r, val, irb);//遍历writeRegs调用storeRegister函数翻译ir，注意这里排除了上面storeOp翻译的ir，否则会重复
    }
} 
```

自己编译
====

*   git clone https://github.com/avast/retdec.git
    
*   `cd retdec`
    
*   `mkdir build && cd build`
    
*   语法`cmake .. -DCMAKE_INSTALL_PREFIX=<path>` -DRETDEC_ENABLE_<component>=ON
    
    cmake ../ -DRETDEC_ENABLE_CAPSTONE2LLVMIRTOOL=ON 只编译 CAPSTONE2LLVMIR 前端，这里是原汁原味一句一句翻译 asm 为 ir 的逻辑，也就是本文讲的
    
    //cmake ../ -DRETDEC_ENABLE_BIN2LLVMIRTOOL=ON 注意这个是之前版本的，现在已经没有 BIN2LLVMIRTOOL 了，只有一个库
    
    cmake ../ -DRETDEC_ENABLE_RETDECTOOL=ON 只编译 RETDECTOOL 前端，也就是之前版本的 bin2llvmir 前端，这里先通过 CAPSTONE2LLVMIR 处理得到的 ir，然后通过很多 pass 对于最初的 ir 进行了分析和优化，其中的到达定值分析和构造西沟分析等都非常的巧妙，值得研究，关键接口函数 retdec::disassemble(po.inputFile, &fs)
    
    ps：这里要从 git 上下载 capstone 与 keystone 与 llvm 相关的库，我下的比较慢 可以修改为国内的源
    
    ```
    git remote set-url --push origin  https://github.com/Hackergeek/architectur
    
    ```
    
*   `make -jN` (`N` 一般设置为核心数 + 1)，然后在 retdec\build\src \ 下面找到可执行文件，像下面这样
    
    retdec-decompiler 是 bin2llvmir2cpp
    
    retdectool 是 bin2llvmir(capstone2llvmir + 多个 pass 优化后)
    
    capstone2llvmirtool 是 capstone2llvmir 原汁原味
    

```
./retdec-decompiler --help
./retdec-decompiler:
Mandatory arguments:
        INPUT_FILE File to decompile.
General arguments:
        [-o|--output FILE] Output file (default: INPUT_FILE.c if OUTPUT_FORMAT is plain, INPUT_FILE.c.json if OUTPUT_FORMAT is json|json-human).
        [-s|--silent] Turns off informative output of the decompilation.
        [-f|--output-format OUTPUT_FORMAT] Output format [plain|json|json-human] (default: plain).
        [-m|--mode MODE] Force the type of decompilation mode [bin|raw] (default: bin).
        [-p|--pdb FILE] File with PDB debug information.
        [-k|--keep-unreachable-funcs] Keep functions that are unreachable from the main function.
        [--cleanup] Removes temporary files created during the decompilation.
        [--config] Specify JSON decompilation configuration file.
        [--disable-static-code-detection] Prevents detection of statically linked code.
Selective decompilation arguments:
        [--select-ranges RANGES] Specify a comma separated list of ranges to decompile (example: 0x100-0x200,0x300-0x400,0x500-0x600).
        [--select-functions FUNCS] Specify a comma separated list of functions to decompile (example: fnc1,fnc2,fnc3).
        [--select-decode-only] Decode only selected parts (functions/ranges). Faster decompilation, but worse results.
Raw or Intel HEX decompilation arguments:
        [-a|--arch ARCH] Specify target architecture [mips|pic32|arm|thumb|arm64|powerpc|x86|x86-64].
                         Required if it cannot be autodetected from the input (e.g. raw mode, Intel HEX).
        [-e|--endian ENDIAN] Specify target endianness [little|big].
                             Required if it cannot be autodetected from the input (e.g. raw mode, Intel HEX).
        [-b|--bit-size SIZE] Specify target bit size [16|32|64] (default: 32).
                             Required if it cannot be autodetected from the input (e.g. raw mode).
        [--raw-section-vma ADDRESS] Virtual address where section created from the raw binary will be placed.

```

retdectool
==========

retdectool 也就是以前的 bin2llvmir 可执行文件，从入口开始学习这个，搞清楚如何通过各种库把汇编转换为 ir，然后通过各种分析优化 pass 得到可读性很强的 ir，main 函数 retdec-master\src\retdectool\retdec.cpp 里, 关键是 disassemble，第一个 string 指针参数表示待处理文件路径 inputPath，第二个生成的 ir 结果，存储在 FunctionSet 类型的 fs 指针，这里可以看一下 retdec::common::Function 的数据结构，存储了函数类型，ir 等有用信息

```
main
{
llvmModuleContextPair disassemble(
        const std::string& inputPath,
        retdec::common::FunctionSet* fs)
 {
    auto context = std::make_unique();
    auto module = createLlvmModule(*context);
 
    config::Config c;
    c.parameters.setInputFile(inputPath);
 
    // Create a PassManager to hold and optimize the collection of passes we
    // are about to build.
    llvm::legacy::PassManager pm;//创建一个PassManager，它的作用是管理pass，我们可以往其中添加很多pass，然后通过run遍历执行所有pass
 
    pm.add(new bin2llvmir::ProviderInitialization(&c));
    //ProviderInitialization继承自modulepass，路径src\bin2llvmir\optimizations\provider_init，执行runonmodule
    pm.add(new bin2llvmir::Decoder());
    //Decoder这个pass就是对capstone2llvmir的进一步封装了
    // Now that we have all of the passes ready, run them.
    pm.run(*module);
 
    fillFunctions(*module, fs);
 
    return LlvmModuleContextPair{std::move(module), std::move(context)};
 }
} 
```

[](#附录：capstone2llvmir目录结构与源码)附录：capstone2llvmir 目录结构与源码
========================================================

<table><thead><tr><th><a href="https://retdec-tc.avast.com/repository/download/Retdec_DoxygenBuild/.lastSuccessful/build/doc/doxygen/html/dir_145127abb956c96185460be5bf09f9bb.html">capstone2llvmir 主目录</a></th><th></th></tr></thead><tbody><tr><td><strong><a href="https://retdec-tc.avast.com/repository/download/Retdec_DoxygenBuild/.lastSuccessful/build/doc/doxygen/html/dir_a1c3be035e2943bb42e5cac140d5edbd.html">arm 分目录</a></strong></td><td></td></tr><tr><td><a href="https://retdec-tc.avast.com/repository/download/Retdec_DoxygenBuild/.lastSuccessful/build/doc/doxygen/html/capstone2llvmir_2arm_2arm_8cpp.html">arm.cpp</a></td><td>ARM implementation of <code>Capstone2LlvmIrTranslator</code> arm 翻译声明</td></tr><tr><td><a href="https://retdec-tc.avast.com/repository/download/Retdec_DoxygenBuild/.lastSuccessful/build/doc/doxygen/html/arm__impl_8h.html">arm_impl.h</a></td><td>ARM implementation of <code>Capstone2LlvmIrTranslator</code> arm 翻译实现</td></tr><tr><td><a href="https://retdec-tc.avast.com/repository/download/Retdec_DoxygenBuild/.lastSuccessful/build/doc/doxygen/html/arm__init_8cpp.html">arm_init.cpp</a></td><td>Initializations for ARM implementation of <code>Capstone2LlvmIrTranslator</code>初始化</td></tr><tr><td><a href="https://retdec-tc.avast.com/repository/download/Retdec_DoxygenBuild/.lastSuccessful/build/doc/doxygen/html/capstone2llvmir_8cpp.html">capstone2llvmir.cpp</a></td><td>Converts bytes to Capstone representation, and Capstone representation to LLVM IR 重要接口声明</td></tr><tr><td><a href="https://retdec-tc.avast.com/repository/download/Retdec_DoxygenBuild/.lastSuccessful/build/doc/doxygen/html/capstone2llvmir__impl_8cpp.html">capstone2llvmir_impl.cpp</a></td><td>Common public interface for translators converting bytes to LLVM IR 重要接口实现</td></tr><tr><td><a href="https://retdec-tc.avast.com/repository/download/Retdec_DoxygenBuild/.lastSuccessful/build/doc/doxygen/html/capstone2llvmir__impl_8h.html">capstone2llvmir_impl.h</a></td><td>Common private implementation for translators converting bytes to LLVM IR</td></tr><tr><td><a href="https://retdec-tc.avast.com/repository/download/Retdec_DoxygenBuild/.lastSuccessful/build/doc/doxygen/html/capstone__utils_8h.html">capstone_utils.h</a></td><td>Utility functions for types, enums, etc. defined in Capstone</td></tr><tr><td><a href="https://retdec-tc.avast.com/repository/download/Retdec_DoxygenBuild/.lastSuccessful/build/doc/doxygen/html/exceptions_8cpp.html">exceptions.cpp</a></td><td>Definitions of exceptions used in capstone2llmvir library</td></tr><tr><td><a href="https://retdec-tc.avast.com/repository/download/Retdec_DoxygenBuild/.lastSuccessful/build/doc/doxygen/html/llvmir__utils_8cpp.html">llvmir_utils.cpp</a></td><td>LLVM IR utilities</td></tr><tr><td><a href="https://retdec-tc.avast.com/repository/download/Retdec_DoxygenBuild/.lastSuccessful/build/doc/doxygen/html/llvmir__utils_8h.html">llvmir_utils.h</a></td><td>LLVM IR utilities</td></tr><tr><td></td></tr></tbody></table>

[第五届安全开发者峰会（SDC 2021）议题征集正式开启！](https://bbs.pediy.com/thread-266645.htm)