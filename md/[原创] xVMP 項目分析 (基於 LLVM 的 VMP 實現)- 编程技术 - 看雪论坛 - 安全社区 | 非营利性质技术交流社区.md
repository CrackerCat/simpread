> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-288800.htm)

> [原创] xVMP 項目分析 (基於 LLVM 的 VMP 實現)

0. 前言
-----

之前在學 LLVM Pass 的時候就在想，是否能用 LLVM Pass 來實現 vmp？想了一陣子沒啥思路就沒再多想了。  
直至看到大佬的這篇文章「[**[原创] 写一个简单的 VMP - 不造轮子，何以知轮之精髓？**](https://bbs.kanxue.com/thread-288435.htm#msg_header_h3_6)」，才發現原來 LLVM Pass 真能寫 VMP，而且十分的契合。  
上述文章參考了 xVMP 這個項目，在閱讀源碼後深感其設計之精妙，令人由衷讚嘆，故而寫下這篇文章與各位靚仔分享，如有寫錯之處，還請指出！！

1. 概述
-----

項目地址：https://github.com/GANGE666/xVMP

xVMP 的流程可分成 3 步：

1.  將 IR 指令轉換成自定義的 ByteCode。
2.  根據模板 ( `xVMPInterpreter.h` ) 來建立 VM 執行環境。
3.  將目標函數置空，替換為我們的 VM 啟動器。

```
virtual bool runOnModule(Module &M)
{
    for (auto Func = M.begin(); Func != M.end(); ++Func)
    {
        Function *F = &*Func;
        if (toObfuscateFunction(this->flag, F, "vmp"))
        {
            // nglog: 忽略具有可變參數的函數
            if (F->isVarArg())
                continue;
 
            // nglog: 重置全局變量
            // 對每個函數應用vmp pass時, 雖然用到都是以下的全局變量, 但在IR層它們的名字是不同的, 因為不用擔心命名沖突的問題
            govm_interpreter = nullptr;
            gv_code_seg = nullptr;
            gv_data_seg = nullptr;
            ip = nullptr;
            data_seg_addr = nullptr;
 
            // nglog: Step1 - IR指令 -> VM ByteCode
            GOVMTranslator *translator = new GOVMTranslator(F);
            translator->run();
            // nglog: Step2 - 建立VM執行環境
            GOVMInterpreter *interpreter = new GOVMInterpreter(F, translator->get_callinst_handler());
            interpreter->run();
            // nglog: Step3 - 替換原函數為VM啟動器
            GOVMModifier *modifier = new GOVMModifier(F, translator->get_gv_value_map());
            modifier->run();
        }
    }
    return true;
}

```

一些全局變量的說明：

*   `gv_code_seg`：llvm ir 層的全局變量，用於保存翻譯後的 ByteCode，它由`vm_code`數組初始化而來。( 下文提到的`gv_code_seg`和`vm_code`可以理解成同樣東西 )
*   `gv_data_seg`：保存了函數運行期間所有用到的數據，記`data_off`為相對於該數組的索引。
*   `ip`：程序計數器，VM 執行時通過它來取指。

2. IR 指令翻譯
----------

### 2.1 初始化

`GOVMTranslator`的構造函數中調用了`init()`進行一些初始化。

```
void init()
{
    setup_callinst_handler();
     
    // encrypt opcode
    init_xorshift32();
}

```

`setup_callinst_handler()`創建了一個 Wrapper 函數，記為`new_call_handler()`，在目前階段仍是一個空殼 (只有一行 func id 判斷的邏輯)，之後會在`handle_callinst()`中填充主邏輯，最後替換模板中的`call_handler()`。

*   `this->callinst_handler`：保存了`new_call_handler()`，類型為`Function*`。
*   `this->callinst_handler_conBBL`：函數體，目前為空。
*   `this->targetfunc_id`：ir 層的 func id。

```
void GOVMTranslator::setup_callinst_handler()
{
    /* nglog: 先初始化一個這樣的空殼函數, 接受一個參數代表函數ID
    void call_handler(uint64_t targetfunc_id)
    {
        // XXX
    }
    */
 
    // collect dispatch function args type
    std::vector FuncTy_args;
    // param: targetfunc_id_value
    FuncTy_args.push_back(Type::getInt64Ty(Mod->getContext()));
 
    // get dispatch function type
    FunctionType *FuncTy = FunctionType::get(
        /*Result=*/Type::getVoidTy(this->Mod->getContext()), // returning void
        /*Params=*/FuncTy_args,
        /*isVarArg=*/false);
 
    Constant *tmp = Function::Create(FuncTy, llvm::GlobalValue::LinkageTypes::InternalLinkage, "vm_interpreter_callinst_dispatch_" + F->getName(), Mod);
    Function *func = cast(tmp);
 
    // create entry BasicBlock
    BasicBlock *entryBB = BasicBlock::Create(func->getContext(), "entryBB", func);
    IRBuilder<> IRBentryBB(entryBB);
 
    // Store params
    Value *target_id_value;
    for (auto arg = func->arg_begin(); arg != func->arg_end(); arg++)
    {
        Value *tmparg = &*arg;
        if (arg == func->arg_begin())
        {
            // targetfunc_id_value
            Value *paramPtr = IRBentryBB.CreateAlloca(Type::getInt64Ty(Mod->getContext()));
            IRBentryBB.CreateStore(tmparg, paramPtr);
            target_id_value = IRBentryBB.CreateLoad(paramPtr);
        }
    }
 
    // create condition basicblock
    BasicBlock *conBBL = BasicBlock::Create(func->getContext(), "conBBL", func);
    IRBentryBB.CreateBr(conBBL);
     
    // traverse Functions and put them in the switch
    this->callinst_handler_curr_idx = 0;
 
    this->callinst_handler = func;
    this->callinst_handler_conBBL = conBBL;
 
    this->targetfunc_id = target_id_value;
} 
```

### 2.2 前置準備

IR 指令翻譯的邏輯在`GOVMTranslator`類的`run()`方法。

若函數返回值不為空，則將`curr_data_offset`加上對應類型的大小，默認偏移`0`用來保存返回值。

注：`curr_data_offset`用來記錄當前`gv_data_seg`的大小。

```
// if return not void, alloca a memory
if (!F->getReturnType()->isVoidTy())
{
    curr_data_offset += modDataLayout->getTypeAllocSize(F->getReturnType());
}

```

遍歷當前函數的參數，並保存到`value_map`中。

( 之後在`handle_inst()`時就是通過`value_map`來找到對應指令的`data_off`，以此進行譯碼 )

```
if (!F->isVarArg())
{
    for (auto arg = F->arg_begin(); arg != F->arg_end(); ++arg)
    {
 
        Value *tmparg = &*arg;
        // value_map[tmparg] = curr_data_offset
        insert_to_value_map(&value_map, tmparg, curr_data_offset);
        curr_data_offset += modDataLayout->getTypeAllocSize(tmparg->getType());
    }
}

```

遍歷當前 Function 的`BB`，每個`BB`對應不同的`opcode_seed`和`vm_code_seed`，前者專門用來加密「操作碼」，後者則是用於對`vm_code`整體的加密 (這部份之後會分析到)。

```
// nglog: 記錄當前BB在gv_code_seg中的索引, 之後處理br指令時會用到basicblock_map
basicblock_map.insert(pair(bb, vm_code.size()));
 
// Opcode seed
// nglog: opcode_seed專門用來加密「操作碼」
uint32_t opcode_seed = opcode_seed_setup();
 
// vm_code seed
// nglog: vm_code_seed用於加密所有vm_code ( 當然也包括「操作碼」 )
uint32_t vm_code_seed = vm_code_seed_setup();
uint32_t currbb_begin = vm_code.size(); 
```

遍歷`BB`中每條`Inst`，同時遍歷每條`Inst`的操作數，若`Inst`中某個操作數是`ConstantExpr`(常量表達式) 類型，則需要把它「拉出來」變成一條獨立的指令，插到`Inst`前面。

所有`Inst`都會被傳入`handle_inst()`進入譯碼。

```
for (auto ins = bbl->begin(); ins != bbl->end(); ins++)
{
 
    Instruction *inst = dyn_cast(ins);
 
    for (unsigned idx = 0; idx < inst->getNumOperands(); idx++)
    {
        // nglog: 假如指令的操作數中有ConstantExpr, 要把它「拉出來」變成一條獨立的指令
        if (ConstantExpr *Op = dyn_cast(inst->getOperand(idx)))
        {
            // we found a ConstantExpr
            // convert ConstantExpr to a equal instruction
            Instruction *const_inst = Op->getAsInstruction();
            const_inst->insertBefore(inst);
            // There is a problem, PHINode must at first instruction of a basicblock, unpack all constantExpr is a potential problem
            // God bless there is not a constantExpr in PHINode
 
            // replace ConstantExpr to a value in inst
            inst->setOperand(idx, const_inst);
 
            handle_inst(const_inst);
        }
    }
    handle_inst(inst);
} 
```

### 2.3 指令譯碼 ( `handle_inst` )

看看`handle_inst()`是怎麼把一條 IR 指令轉譯成自己的 ByteCode。

`AllocaInst`指令：( 例如 ⇒ `%ptr = **alloca** i32` )

*   由於`AllocaInst`的功能是在棧上分配一片內存，因此要預留`AllocaInst_Res`和`AllocaInst_alloca_area`。
*   `AllocaInst_Res`是為了之後 load/store 時能找到目標。
*   `AllocaInst_alloca_area`用來模擬分配的內存空間。
*   調用`GET_PACK_VALUE()`來打包當前指令。

```
if (AllocaInst *inst = dyn_cast(ins))
{
    // alloca memory for AllocaInst_Res and AllocaInst_alloca_area
 
    // nglog: 預留位置，用於保存AllocaInst的返回值
    // AllocaInst_Res
    int pointer_offset = curr_data_offset;
    // nglog: 記錄到value_map中, 之後load/store時才知道要訪問哪裡
    insert_to_value_map(&value_map, inst, curr_data_offset);
    int res_size = modDataLayout->getTypeAllocSize(inst->getType());
    curr_data_offset += res_size;
 
    // nglog: 打包指令
    std::vector packed_res = GET_PACK_VALUE(inst);
 
    // nglog: 預留位置, 用於模擬AllocaInst分配的棧空間
    // AllocaInst_alloca_area
    int area_offset = curr_data_offset;
    int alloca_size = modDataLayout->getTypeAllocSize(inst->getAllocatedType());
    curr_data_offset += alloca_size;
 
    // nglog: 構建AllocaInst的字節碼
    std::vector hex_code;
    ins_to_hex(hex_code, pack_op(ALLOCA_OP), packed_res, pack(area_offset, POINTER_SIZE));
    vm_code.insert(vm_code.end(), hex_code.begin(), hex_code.end());
 
} 
```

`GET_PACK_VALUE`是個宏，最終調用的是`packValue()`。

首先構造`packType = {size, TypeID}` (占 2B)，然後判斷傳入的 Value 是否常量，在上述情況下顯然不是。

然後構建`packed = pack((*value_map)[value], POINTER_SIZE)`，在上述情況下就是把`AllocaInst_Res`轉成 8B 的字節流。

`packValue()`在遇到全局變量時會將其一併保存到`gv_value_map`中，留到之後處理 (後面會分析到)。

```
#define GET_PACK_VALUE(value) (packValue(value, &value_map))
// pack type to a vector(2)
// {size, TypeID}
std::vector type_to_hex(Type *type)
{
    std::vector res;
    res.push_back(modDataLayout->getTypeAllocSize(type));
    res.push_back(type->getTypeID());
    return res;
}
 
// pack a value
std::vector packValue(Value *value, std::map *value_map)
{
    std::vector res;
    std::vector packed;
    std::vector packType = type_to_hex(value->getType());
    if (ConstantData *CD = dyn_cast(value))
    {
        packed = pack_const_value(value);
    }
    else
    {
        // if value not in map
        if (value_map->find(value) == value_map->end())
        {
            // check value is not a GlobalVariable
            if (GlobalVariable *gv = dyn_cast(value))
            {
                // is a GlobalVariable and not in value_map
                // put it into value_map
                insert_to_value_map(value_map, value, curr_data_offset);
 
                // also put it into gv_value_map
                gv_value_map.insert(pair(gv, curr_data_offset));
 
                int res_size = modDataLayout->getTypeAllocSize(gv->getType());
                curr_data_offset += res_size;
            }
            else
            {
                assert(value_map->find(value) != value_map->end());
            }
        }
 
        packed = pack((*value_map)[value], POINTER_SIZE);
        // variableï¼Œpacktype->TypeID=0
        packType[1] = 0;
    }
 
    res.insert(res.end(), packType.begin(), packType.end());
    res.insert(res.end(), packed.begin(), packed.end());
 
    return res;
} 
```

最終`alloca`指令會被轉譯成如下字節流，保存在`gv_code_seg`中。

```
// | 字節碼 | 打包的返回值 | 申請的空間 |
 | | 
```

同時看看 VM 解釋器是怎樣解析上述字節流的。

`get_byte_code()`會返回`gv_code_seg[ip++]`，但`alloca`的返回值固定是 8 字節的變量，因此`var_size`和`var_type`都沒有用。

`unpack_code()`會以小端形式從`gv_code_seg`讀取指定大小的值。`data_seg_addr`是運行時`gv_data_seg`的實際地址，最後那句代碼會把`data_seg_addr + area_offset`這個值保存到`data_seg_addr + var_offset`這個地址。

```
#ifdef IS_INLINE_FUNC
    __inline__ __attribute__((always_inline))
#endif
void alloca_handler() {
    // size and type of pointer is useless
    uint8_t var_size = get_byte_code();
    uint8_t var_type = get_byte_code();
 
    // get pointer var offset
    uint64_t var_offset = unpack_code(POINTER_SIZE);
 
    // get alloca area offset
    uint64_t area_offset = unpack_code(POINTER_SIZE);
 
    // store area virtual address to var
    // set_var(var_offset, POINTER_SIZE, data_seg_addr+area_offset);
    pack_store_addr(data_seg_addr+var_offset, data_seg_addr+area_offset, var_size);
}

```

`LoadInst`指令：( 例子 ⇒ `%val = load i32, i32* %ptr` )

*   `return`：`LoadInst`的返回值。
*   `PointerOperand`：`LoadInst`中第`0`個操作數，即某條`alloca`指令的返回值。

```
else if (LoadInst *inst = dyn_cast(ins))
{
 
    // return
    int res_offset = curr_data_offset;
    insert_to_value_map(&value_map, inst, curr_data_offset);
    int res_size = modDataLayout->getTypeAllocSize(inst->getType());
    curr_data_offset += res_size;
 
    std::vector packed_res = GET_PACK_VALUE(inst);
 
    // nglog: LoadInst指令的getPointerOperand()會返回「指針操作數」(即某個AllocaInst的返回值)
    // PointerOperand
    std::vector packed_pointer_operand = GET_PACK_VALUE(inst->getPointerOperand());
 
    std::vector hex_code;
    ins_to_hex(hex_code, pack_op(LOAD_OP), packed_res, packed_pointer_operand);
    vm_code.insert(vm_code.end(), hex_code.begin(), hex_code.end());
} 
```

`StoreInst`指令：( 例子 ⇒ **`store** i32 3, ptr %ptr` )

*   `StoreInst`沒有返回值。
*   `ValueOperand`和`PointerOperand`可從`StoreInst`的例子明顯看出。

```
else if (StoreInst *inst = dyn_cast(ins))
{
 
    // ValueOperand
    std::vector packed_value_operand = GET_PACK_VALUE(inst->getValueOperand());
 
    // PointerOperand
    std::vector packed_pointer_operand = GET_PACK_VALUE(inst->getPointerOperand());
 
    std::vector hex_code;
    ins_to_hex(hex_code, pack_op(STORE_OP), packed_value_operand, packed_pointer_operand);
    vm_code.insert(vm_code.end(), hex_code.begin(), hex_code.end());
 
} 
```

`GetElementPtrInst`指令：( 例子 ⇒ `%elem_ptr = getelementptr i32, ptr %array_ptr, i32 2` )

*   收集了所有索引，但只會用只後一個索引。
*   分成`struct`和`array`兩種情況，前者要計算字段偏移，後者直接記錄索引值即可。

```
else if (GetElementPtrInst *inst = dyn_cast(ins))
{
    int res_offset = curr_data_offset;
    insert_to_value_map(&value_map, inst, curr_data_offset);
    int res_size = modDataLayout->getTypeAllocSize(inst->getType());
    curr_data_offset += res_size;
 
    std::vector packed_res = GET_PACK_VALUE(inst);
 
    std::vector packed_ptr = GET_PACK_VALUE(inst->getPointerOperand());
 
    // get indices
    // but only consider last indice
    std::vector indices;
    for (auto curr_idx = inst->idx_begin(); curr_idx != inst->idx_end(); curr_idx++)
    {
        indices.push_back(*curr_idx);
    }
 
    // GEP type
    // {0, 0}: structure value is offset
    // {x, x}: array, value is offset
    Type *srcType = inst->getSourceElementType();
    std::vector gep_type;
    std::vector packed_value;
    if (dyn_cast(srcType))
    {
        // is struct type
        StructType *st = dyn_cast(srcType);
        gep_type = {0, 0};
        ConstantInt *CI = dyn_cast(indices[indices.size() - 1]); // last indice
        int element_idx = CI->getSExtValue();                                 // const value to int
        int curr_element_offset = 0;
        for (int i = 0; i < element_idx; i++)
        { // calc the offset between curr_element and struct_begin
            curr_element_offset += modDataLayout->getTypeAllocSize(st->getElementType(i));
        }
 
        // Construct const-offset manually
        packed_value = {0, 0};
    std:
        vector tmp = pack(curr_element_offset, POINTER_SIZE);
        packed_value.insert(packed_value.end(), tmp.begin(), tmp.end());
    }
    else
    {
        // is array type
        gep_type = type_to_hex(inst->getResultElementType());
        packed_value = GET_PACK_VALUE(indices[indices.size() - 1]);
    }
 
    std::vector hex_code;
    ins_to_hex(hex_code, pack_op(GEP_OP), gep_type, packed_res, packed_ptr, packed_value);
    vm_code.insert(vm_code.end(), hex_code.begin(), hex_code.end());
 
} 
```

為什麼只指最後一個索引就可以？看一個實際例子。

```
int test16(int arr[3][2]) {
    return arr[2][0];
}

```

在`-O0`優化下，上述代碼對應的 llvm ir 如下。

可以看到在取`arr[2][0]`時分成了 2 條`gep`指令，第 1 條 gep 指令返回`&arr[2]`。

而第 2 條`gep`指令有兩個索引，第 1 個 0 代表`&arr[2]`( 因為`<ty>`是`[2 x i32]` )，第 2 個 0 代表`&arr[2][0]`。

由此可見在`-O0`優化下會把多層的指針操作拆分到多條`gep`指令來執行，單條`gep`指令最多只有 2 個索引 (不確定是否有例外？)，只有最後那個索引才是「有用」的。

```
; Function Attrs: mustprogress noinline nounwind optnone uwtable
define dso_local noundef i32 @_Z6test16PA2_i(ptr noundef %arr) #4 {
entry:
  %arr.addr = alloca ptr, align 8
  store ptr %arr, ptr %arr.addr, align 8
  %0 = load ptr, ptr %arr.addr, align 8
  %arrayidx = getelementptr inbounds [2 x i32], ptr %0, i64 2
  %arrayidx1 = getelementptr inbounds [2 x i32], ptr %arrayidx, i64 0, i64 0
  %1 = load i32, ptr %arrayidx1, align 4
  ret i32 %1
}


```

然後看看`-O1`優化，它把 2 條`gep`指令合併成 1 條，類型被修改為`i8`，索引只有 1 個`16`，相當於返回`((uint8_t*)arr + 16)`，它等價於`&arr[2][0]`。

```
; Function Attrs: mustprogress nofree norecurse nosync nounwind willreturn memory(argmem: read) uwtable
define dso_local noundef i32 @_Z6test16PA2_i(ptr nocapture noundef readonly %arr) #7 {
entry:
  %arrayidx = getelementptr inbounds i8, ptr %arr, i64 16
  %0 = load i32, ptr %arrayidx, align 4, !tbaa !5
  ret i32 %0
}


```

`BranchInst`指令：( 例子 ⇒ `br i1 %cmp, label %if.then, label %if.end` )

*   分成有條件和無條件跳轉，`0`代表無條件，`1`代表有條件。
*   跳轉地址暫時置空 ( `vm_code`傳入`padding`來占位 )，跳轉信息先保存在`br_map`，在遍歷完整個函數後統一處理。

```
else if (BranchInst *inst = dyn_cast(ins))
{
    // Construct code_hex in here manually
    std::vector hex_code;
    hex_code = pack_op(BR_OP);
    std::vector padding = pack(0, POINTER_SIZE);
 
    if (inst->isUnconditional())
    {
        hex_code.push_back(0);
        // errs() << vm_code.size()+2 << "\n";
        br_map.push_back(pair(vm_code.size() + hex_code.size(), inst->getSuccessor(0))); // fill after traverse whole function
        hex_code.insert(hex_code.end(), padding.begin(), padding.end());
    }
    else
    {
        hex_code.push_back(1);
        // condition
        std::vector pack_condition = packValue(inst->getCondition(), &value_map);
        hex_code.insert(hex_code.end(), pack_condition.begin(), pack_condition.end());
 
        // errs() << vm_code.size()+2 << "\n";
        br_map.push_back(pair(vm_code.size() + hex_code.size(), inst->getSuccessor(0)));
        hex_code.insert(hex_code.end(), padding.begin(), padding.end());
        // errs() << vm_code.size()+2+POINTER_SIZE << "\n";
        br_map.push_back(pair(vm_code.size() + hex_code.size(), inst->getSuccessor(1)));
        hex_code.insert(hex_code.end(), padding.begin(), padding.end());
    }
 
    vm_code.insert(vm_code.end(), hex_code.begin(), hex_code.end());
 
} 
```

處理邏輯很簡單，就是根據`br_map`信息回填跳轉偏移 ( 相對於`gv_code_seg` )。

```
// fill br map
for (auto it = br_map.rbegin(); it != br_map.rend(); it++)
{
    int code_pos = it->first;
    BasicBlock *target_bb = it->second;
     
    std::vector bb_addr = pack(basicblock_map[target_bb], POINTER_SIZE);
    std::copy(bb_addr.begin(), bb_addr.end(), vm_code.begin() + code_pos);
} 
```

`CallInst`指令：( 例子 ⇒ `call void @decryptString.2(ptr %9)` )

*   為每個 call 分配一個 func id，用`callinst_map`記錄當前`CallInst`與 func id 的對應關係。

```
else if (CallInst *inst = dyn_cast(ins))
{
 
    // current function id
    long long curr_func_id = this->callinst_handler_curr_idx++;
 
    std::vector packed_funcid = pack(curr_func_id, POINTER_SIZE);
 
    // check if this callsite return a void
    std::vector packed_res;
    if (inst->getType() != Type::getVoidTy(this->Mod->getContext()))
    {
        // return a value
        int res_offset = curr_data_offset;
        insert_to_value_map(&value_map, inst, curr_data_offset);
        int res_size = modDataLayout->getTypeAllocSize(inst->getType());
        curr_data_offset += res_size;
 
        packed_res = GET_PACK_VALUE(inst);
    }
 
    // construct hex code
    std::vector hex_code;
    ins_to_hex(hex_code, pack_op(Call_OP), packed_funcid);
     
    vm_code.insert(vm_code.end(), hex_code.begin(), hex_code.end());
 
    // handle_callinst(inst, curr_func_id);
    callinst_map.insert(std::pair(inst, curr_func_id));
} 
```

之後 (所有指令完成譯碼後) 會調用`handle_callinst()`來專門處理`CallInst`。

```
for (auto p : callinst_map)
{
    handle_callinst(p.first, p.second);
}

```

### 2.4 處理函數調用 ( `handle_callinst` )

`handle_callinst()`就是在不斷完善`new_call_handler()`，最後`new_call_handler`會替換模板中的`call_handler()`。

首先處理`CallInst`指令的參數，分兩種情況：

1.  若參數是常量，則直接保存到`target_func_args`。
2.  否則要從`value_map`獲取參數的`data_off` → 然後構建`gep`指令指向該參數 → 類型轉換 → 構建 IR 指令讀取成`Value*` → 保存到`target_func_args` 。

```
// nglog: 指向new_call_handler的待插入點
IRBuilder<> IRBcon(this->callinst_handler_conBBL);
 
// firstly, we need to unpack function args
std::vector target_func_args;
for (unsigned idx = 0; idx < inst->getNumArgOperands(); idx++)
{
    Value *currarg = inst->getArgOperand(idx);
     
    // nglog: 常量直接保存在target_func_args
    // if value is a constant, use it directly
    if (ConstantData *CD = dyn_cast(currarg))
    {
        target_func_args.push_back(currarg);
        continue;
    }
 
    unsigned curroffset = value_map[currarg];
 
    // nglog: 類型轉換 -> 構建IR指令加載參數值 -> 保存在target_func_args
    // construct load
    ConstantInt *Zero = ConstantInt::get(Type::getInt64Ty(Mod->getContext()), 0);
    Value *offset_value = ConstantInt::get(Type::getInt64Ty(Mod->getContext()), curroffset);
    Value *gepinst = IRBcon.CreateGEP(gv_data_seg, {Zero, offset_value}, "");
 
    // convert gep from i8* to value->getType() *
    PointerType *target_ptr_type = PointerType::get(currarg->getType(), cast(gepinst->getType())->getAddressSpace());
    Value *ptr = IRBcon.CreatePointerCast(gepinst, target_ptr_type);
 
    // load from gv_data_seg
    Value *arg = IRBcon.CreateLoad(ptr);
 
    target_func_args.push_back(arg);
} 
```

然後創建了一個新的`BasicBlock`來構建`CallInst`指令，`CallInst`指令可分成直接調用和間接調用。

```
// nglog: 創建一個新的BB來構建CallInst
// secondly, we create a new basic block to construct callinst
BasicBlock *callFunction = BasicBlock::Create(Mod->getContext(), "callFunction_" + to_string(this->callinst_handler_curr_idx), this->callinst_handler);
IRBuilder<> IRBcallFunction(callFunction);
 
Value *resultValue;
 
if (!inst->isIndirectCall())
{
    // direct call
    Function *callee = inst->getCalledFunction();
    // call replace function
    resultValue = IRBcallFunction.CreateCall(callee->getFunctionType(), callee,
                                             ArrayRef(target_func_args));
}
else
{
    // indirect call
 
    // nglog: getCalledValue() 返回call指令調用的東西 (並非間接調用特有)
    // 1. %call = call i32 @foo(i32 %arg)               (就是 @foo)
    // 2. %fp = load i32 (i32)*, i32 (i32)** %fp_ptr
    //      %call = call i32 %fp(i32 %arg)              (就是%fp)
    Value *called_value = inst->getCalledValue();
    unsigned called_value_offset = value_map[called_value];
 
    // load value from gv_data_seg
    ConstantInt *Zero = ConstantInt::get(Type::getInt64Ty(Mod->getContext()), 0);
    Value *offset_value = ConstantInt::get(Type::getInt64Ty(Mod->getContext()), called_value_offset);
    Value *gepinst = IRBcallFunction.CreateGEP(gv_data_seg, {Zero, offset_value}, "");
 
    // convert gep from i8* to value->getType() *
    PointerType *target_ptr_type = PointerType::get(called_value->getType(), cast(gepinst->getType())->getAddressSpace());
    Value *ptr = IRBcallFunction.CreatePointerCast(gepinst, target_ptr_type);
 
    // load from gv_data_seg
    Value *value = IRBcallFunction.CreateLoad(ptr);
 
    // indirect call
    resultValue = IRBcallFunction.CreateCall(value, ArrayRef(target_func_args));
} 
```

返回值不為 void 時，需要構建 ir 指令把返回值保存到`gv_data_seg`指定位置。

```
// if return not void, store it to gv_data_seg
if (inst->getType() != Type::getVoidTy(this->Mod->getContext()))
{
    unsigned result_value_offset = value_map[inst];
 
    // load value from gv_data_seg
    ConstantInt *Zero = ConstantInt::get(Type::getInt64Ty(Mod->getContext()), 0);
    Value *offset_value = ConstantInt::get(Type::getInt64Ty(Mod->getContext()), result_value_offset);
    Value *gepinst = IRBcallFunction.CreateGEP(gv_data_seg, {Zero, offset_value}, "");
 
    // convert gep from i8* to value->getType() *
    PointerType *target_ptr_type = PointerType::get(resultValue->getType(), cast(gepinst->getType())->getAddressSpace());
    Value *ptr = IRBcallFunction.CreatePointerCast(gepinst, target_ptr_type);
 
    // store
    IRBcallFunction.CreateStore(resultValue, ptr);
}
 
// Create Return
IRBcallFunction.CreateRetVoid(); 
```

最後的賦值是為了下一輪的`handle_callinst()`做準備。

```
// compare and jmp
BasicBlock *falseconBBL = BasicBlock::Create(Mod->getContext(), "falseconBBL", this->callinst_handler);
 
// nglog: 構建類似switch結構來進行函數分發
Value *currfunc_id = ConstantInt::get(Type::getInt64Ty(Mod->getContext()), curr_func_id);
Value *condition = IRBcon.CreateICmpEQ(this->targetfunc_id, currfunc_id);
IRBcon.CreateCondBr(condition, callFunction, falseconBBL);
this->callinst_handler_conBBL = falseconBBL;

```

最後一次`handle_callinst()`後，會調用`finish_callinst_handler()`，以保證函數的完整性。

```
void GOVMTranslator::finish_callinst_handler()
{
    // default branch
    IRBuilder<> IRBcon(callinst_handler_conBBL);
 
    // Create Return
    IRBcon.CreateRetVoid();
}

```

最終的`new_call_handler()`大概會是類似這樣的結構：

```
void new_call_handler(uint64_t targetfunc_id) {
        uint64_t func_id = targetfunc_id;
        Arg0Type a0 = gv_data_seg[a];
        Arg1Type a1 = gv_data_seg[b];
        if (func_id == 0) {
                return func0(a0, a1);
        }
        // ...
        if (func_id == 1) {
                return func1();
        }
         
        // do noting....
         
}

```

### 2.5 剩餘部份

在`handle_callinst()`之前，還調用了`encrypt_vm_code()`和`construct_gv()`。

```
encrypt_vm_code();
 
construct_gv();
 
// handle callinst
for (auto p : callinst_map)
{
    handle_callinst(p.first, p.second);
}

```

`encrypt_vm_code()`會以 BB 為單位加密`vm_code`。( 上面說到每個 BB 對應不同的`vm_code_seed`，被記錄在`vm_code_seed_map`中 )。

```
void encrypt_vm_code()
{
    for (auto p : vm_code_seed_map)
    {
        uint32_t vm_code_seed = p.first;
        for (uint32_t addr = p.second.first; addr < p.second.second; addr++)
        {
            vm_code[addr] ^= (xorshift32(&vm_code_seed) & 0xFF);
        }
    }
}

```

`construct_gv()`用於構建 ir 全局變量，如通過`vm_code`構建`gv_code_seg`全局變量。

這裡構建的 ir 全局變量，是為了之後替換模板中對應的全局變量。

```
void GOVMTranslator::construct_gv()
{
    // construct code global array from vm_code
 
    // set Initializer for gv_code_seg
    ArrayRef code_seg_arrayref(vm_code);
    Constant *code_seg_init = ConstantDataArray::get(Mod->getContext(), code_seg_arrayref);
 
    ArrayType *code_seg_type = ArrayType::get(IntegerType::get(Mod->getContext(), 8), vm_code.size());
    // ArrayType * code_seg_type = ArrayType::get(IntegerType::get(Mod->getContext(), 8), VM_CODE_SEG_SIZE);
    // global
    gv_code_seg = new GlobalVariable(
        /*Module=*/*Mod,
        /*Type=*/code_seg_type,
        /*isConstant=*/true,
        /*Linkage=*/GlobalValue::InternalLinkage,
        /*Initializer=*/code_seg_init, // has initializer, specified
                                       // below
        /* + F->getName());
 
    // construct data global array
    std::vector data_seg_vector(curr_data_offset);
    // std::vector data_seg_vector(VM_DATA_SEG_SIZE);
    ArrayRef data_seg_arrayref(data_seg_vector);
    Constant *data_seg_init = ConstantDataArray::get(Mod->getContext(), data_seg_arrayref);
    ArrayType *data_seg_type = ArrayType::get(IntegerType::get(Mod->getContext(), 8), curr_data_offset);
    // ArrayType * data_seg_type = ArrayType::get(IntegerType::get(Mod->getContext(), 8), VM_DATA_SEG_SIZE);
    // global
    gv_data_seg = new GlobalVariable(
        /*Module=*/*Mod,
        /*Type=*/data_seg_type,
        /*isConstant=*/false,
        /*Linkage=*/GlobalValue::InternalLinkage,
        /*Initializer=*/data_seg_init, // has initializer, specified
                                       // below
        /* + F->getName());
 
    // ip
    Constant *ip_initGV = ConstantInt::get(Type::getInt32Ty(Mod->getContext()), 0);
    ip = new GlobalVariable(*Mod, Type::getInt32Ty(Mod->getContext()),
                            false, GlobalValue::InternalLinkage,
                            ip_initGV, "ip_" + F->getName());
 
    // data_seg_addr
    Constant *data_seg_addr_initGV = ConstantInt::get(Type::getInt64Ty(Mod->getContext()), 0);
    data_seg_addr = new GlobalVariable(*Mod, Type::getInt64Ty(Mod->getContext()),
                                       false, GlobalValue::InternalLinkage,
                                       data_seg_addr_initGV, "data_seg_addr_" + F->getName());
 
    // code_seg_addr
    Constant *code_seg_addr_initGV = ConstantInt::get(Type::getInt64Ty(Mod->getContext()), 0);
    code_seg_addr = new GlobalVariable(*Mod, Type::getInt64Ty(Mod->getContext()),
                                       false, GlobalValue::InternalLinkage,
                                       code_seg_addr_initGV, "code_seg_addr_" + F->getName());
} 
```

3. 建立 VM 執行環境
-------------

相關邏輯在`GOVMInterpreter::run()`中。

### 3.1 模板加載

調用了`llvm_parse_bitcode_from_string()`來加載模板。

```
void GOVMInterpreter::run()
{
     
    Module *interpreter_module = llvm_parse_bitcode_from_string();
    assert(interpreter_module);
        // ...
}

```

`llvm_parse_bitcode_from_string()`實現如下，`binary_ir_length`是`.bc`字節碼的長度，`binary_ir_vector`是個 string 數組，每個元素都是`.bc`字節碼的字符串形式。

通過`parseIR()`來解析`.bc`字節碼，最後返回一個`Module*`。

```
Module *llvm_parse_bitcode_from_string()
{
    binary_ir.resize(binary_ir_length);
    int binary_ir_idx = 0;
    for (auto s : binary_ir_vector)
    {
        for (int i = 0; i < s.size(); i++)
        {
            binary_ir[binary_ir_idx++] = s[i];
        }
    }
 
    StringRef str_ref(binary_ir);
    MemoryBufferRef buf_ref = MemoryBufferRef(str_ref, str_ref);
 
    SMDiagnostic Err;
    LLVMContext *LLVMCtx = &Mod->getContext();
    unique_ptr M = parseIR(buf_ref, Err, *LLVMCtx);
    return M.release();
} 
```

### 3.2 模板修正

把模板中的一些全局變量替換為上面`construct_gv()`構建的那些全局變量 ( `opcode_xorshift32_state`和`vm_code_state`是在`GOVMInterpreter`的構造函數裡構建的 )。

把模板中的`call_handler()`替換為上面的`new_call_handler()`。

```
// nglog: 把模板中的全局變量掉換為已經初始化好的全局變量
std::vector gv_list = {"ip", "data_seg_addr", "code_seg_addr", "opcode_xorshift32_state", "vm_code_state"};
std::vector new_gv_list = {ip, data_seg_addr, code_seg_addr, opcode_xorshift32_state, vm_code_state};
for (unsigned i = 0; i < gv_list.size(); i++)
{
    errs() << "[*] Replacing GlobalVariable: " << *new_gv_list[i] << "\n";
    GlobalVariable *old_gv = interpreter_module->getGlobalVariable(gv_list[i]);
    // GlobalVariable *new_gv = Mod->getGlobalVariable(name);
    GlobalVariable *new_gv = new_gv_list[i];
 
    errs() << "old_gv->getType(): " << *old_gv->getType() << "\n";
    errs() << "new_gv->getType(): " << *new_gv->getType() << "\n";
 
    old_gv->replaceAllUsesWith(new_gv);
}
 
// nglog: 模板中的call_handler是個空函數, 這裡替換為實現好的
Function *old_func = interpreter_module->getFunction("call_handler");
errs() << "[*] Replacing function: " << old_func->getName().str() << "\n";
old_func->replaceAllUsesWith(callinst_handler); 
```

然後 clone 模板中的`vm_interpreter()`到當前 Module，並重命名成`vm_interpreter_<func_name>`來防止命名沖突。

然後將所有`vm_interpreter()`調用改為`vm_interpreter_<func_name>()`。

注：「模板」中其餘函數若不聲明為 inline，則同樣需要進行上述處理。

```
// clone functions
for (auto Func = interpreter_module->begin(); Func != interpreter_module->end(); ++Func)
{
 
    Function *fun = &*Func;
 
    // nglog: 在定義了IS_INLINE_FUNC宏時, 這裡僅是在判斷"vm_interpreter"函數
    // 因為其他handler函數被聲明為inline, 不會有名字沖突的問題?
    if (is_interpreter_function(fun))
    {
        // nglog: 把模板裡的vm_interpreter()插入到當前Module
        Constant *tmp = Mod->getOrInsertFunction(fun->getName().str(), fun->getFunctionType());
        Function *NewF = cast(tmp);
        NewF->setLinkage(llvm::GlobalValue::LinkageTypes::InternalLinkage);
 
        // nglog: 這步對於vm_interpreter()或許不是必要的, 因為該函數沒有參數
        // setup VMap
        ValueToValueMapTy VMap;
        SmallVector returns;
 
        Function::arg_iterator DestI = NewF->arg_begin();
 
        for (const Argument &I : fun->args())
            if (VMap.count(&I) == 0)
            {                                // Is this argument preserved?
                DestI->setName(I.getName()); // Copy the name over...
                VMap[&I] = &*DestI++;        // Add mapping to VMap
            }
 
        CloneFunctionInto(NewF, fun, VMap, true, returns);
         
        // nglog: 每個被vmp的函數都會對應一個vm_interpreter函數
        // 因此會存在多個vm_interpreter(), 重命名成vm_interpreter_來防止沖突
        // set a new name
        NewF->setName(fun->getName() + "_" + F->getName());
 
        errs() << "[*] Function: " << fun->getName().str() << " Clone finished!\n";
 
        // collect all references
        std::vector F_users;
        for (User *U : fun->users())
        {
            if (CallInst *CI = dyn_cast(U))
            {
                F_users.push_back(CI);
            }
        }
 
        // replace references
        for (CallInst *CI : F_users)
        {
            errs() << "[*] Replacing references: " << *CI << "\n";
            CI->setCalledFunction(NewF);
        }
    }
} 
```

最後把所有無調用的函數刪掉。

`govm_interpreter`保存了上面 clone 出來的`vm_interpreter_<func_name>`函數。

```
// remove all function of interpreter_module
while (true)
{
    bool flag = true;
    for (auto Func = interpreter_module->begin(), Funcend = interpreter_module->end(); Func != Funcend; ++Func)
    {
 
        Function *fun = dyn_cast(&*Func);
 
        if (fun->use_empty())
        {
            errs() << "[*] Removing function: " << fun->getName().str() << "\n";
            flag = false;
            fun->eraseFromParent();
            break;
        }
    }
    if (flag)
        break;
}
 
govm_interpreter = Mod->getFunction("vm_interpreter_" + F->getName().str()); 
```

4. 原函數替換
--------

相關邏輯在`GOVMModifier::run()`中。

首先清空原函數體，創建`body_bbl`和`ret_bbl`。

```
// remove old function body
llvm::GlobalValue::LinkageTypes linkagetype = F->getLinkage();
F->deleteBody();
F->setLinkage(linkagetype);
 
// create new basicblock
BasicBlock *ret_bbl = BasicBlock::Create(this->Mod->getContext(), "ret", F);
BasicBlock *body_bbl = BasicBlock::Create(this->Mod->getContext(), "body", F, ret_bbl);
IRBuilder<> irbuilder(body_bbl);

```

將原函數參數保存在`args_map`。

```
// collect all callinst args
std::vector> args_map;
int arg_offset = 0;
// if return not void
if (!F->getReturnType()->isVoidTy())
{
    arg_offset += modDataLayout->getTypeAllocSize(F->getReturnType());
}
 
for (auto arg = F->arg_begin(); arg != F->arg_end(); arg++)
{
    Value *tmparg = &*arg;
 
    Value *paramPtr = irbuilder.CreateAlloca(tmparg->getType());
    irbuilder.CreateStore(tmparg, paramPtr);
    Value *currvalue = irbuilder.CreateLoad(paramPtr);
 
    // insert to args_map
    args_map.push_back(pair(currvalue, arg_offset));
    arg_offset += modDataLayout->getTypeAllocSize(tmparg->getType());
 
} 
```

然後遍歷`gv_value_map`，構建 ir 指令，把原函數所用到的全局變量保存到`gv_data_seg`中。

**具體是把「編譯期才能確定的全局變量地址」複製到 VM 的 data segment 中，讓解釋器在執行期也能靠 offset 拿到對應的全局變量指針。**

```
for (auto p : *gv_value_map)
{
    GlobalVariable *gv = p.first;
    int offset = p.second;
 
    errs() << "[*] storing GlobalVariable: " << *gv << "\t" << offset << "\n";
 
    // convert pointer to int64
    Value *gv_addr_int = irbuilder.CreatePtrToInt(gv, Type::getInt64Ty(Mod->getContext()));
 
    // create gep: data_seg + offset
    ConstantInt *Zero = ConstantInt::get(Type::getInt64Ty(Mod->getContext()), 0);
    Value *offset_value = ConstantInt::get(Type::getInt64Ty(Mod->getContext()), offset);
    Value *gepinst = irbuilder.CreateGEP(gv_data_seg, {Zero, offset_value}, "");
 
    // convert gep i8* to i64*
    PointerType *target_ptr_type = PointerType::get(gv_addr_int->getType(), cast(gepinst->getType())->getAddressSpace());
    Value *ptr = irbuilder.CreatePointerCast(gepinst, target_ptr_type);
 
    // store gv_addr_int to data_seg+offset
    irbuilder.CreateStore(gv_addr_int, ptr);
} 
```

然後構建 ir 指令，將原函數參數保存到`gv_data_seg`。

```
// store args to data_seg
for (auto p : args_map)
{
    Value *value = p.first;
    int offset = p.second;
 
    errs() << "[*] storing value: " << *value << "\t" << offset << "\n";
 
    // GEP get ptr point to offset
    ConstantInt *Zero = ConstantInt::get(Type::getInt64Ty(F->getContext()), 0);
    Value *const_curr_value_offset = ConstantInt::get(Type::getInt64Ty(F->getContext()), offset);
    Value *gepinst = irbuilder.CreateGEP(gv_data_seg, {Zero, const_curr_value_offset}, "");
 
    // cast gep_ptr to value->type
    PointerType *target_ptr_type = PointerType::get(value->getType(), cast(gepinst->getType())->getAddressSpace());
    Value *ptr = irbuilder.CreatePointerCast(gepinst, target_ptr_type);
 
    // store value to data_seg+offset
    irbuilder.CreateStore(value, ptr);
} 
```

這裡與上面同理，都是把「編譯期才能確定的地址」轉成 64bit 數值，寫進`data_seg_addr`和`code_seg_addr`。

```
// store gv_data_seg and gv_code_seg to data_seg_addr, code_seg_addr
Value *data_seg_ptr2int = irbuilder.CreatePtrToInt(gv_data_seg, Type::getInt64Ty(Mod->getContext()));
irbuilder.CreateStore(data_seg_ptr2int, data_seg_addr);
Value *code_seg_ptr2int = irbuilder.CreatePtrToInt(gv_code_seg, Type::getInt64Ty(Mod->getContext()));
irbuilder.CreateStore(code_seg_ptr2int, code_seg_addr);

```

模板中所有對`gv_data_seg`和`gv_code_seg`的操作都是通過`data_seg_addr`和`code_seg_addr`，它們分別代表`gv_data_seg`和`gv_code_seg`運行時的實際地址。

( 若僅通過`gv_data_seg + offset`的形式來讀 / 寫數據，在遇到指針操作時會有問題，後面會提到 )

```
uint64_t
unpack_data(uint64_t offset, int size)
{
    uint64_t res = 0;
 
    for (int i = 0; i < size; i++)
    {
        // must add (uint64_t), or overflow int32
        res |= (uint64_t)((uint8_t *)data_seg_addr)[offset++] << (8 * i);
    }
 
    return res;
}

```

最後插入`govm_interpreter`的函數調用，並根據原函數是否有返回值做對應的處理。

注：返回值默認保存在`gv_data_seg[0]`的位置。

```
CallInst *resultvalue = irbuilder.CreateCall(govm_interpreter);
 
if (!F->getReturnType()->isVoidTy())
{
    // unsigned return_value_size = modDataLayout->getTypeAllocSize(F->getReturnType());
    // load return value from data_seg
 
    // GEP get ptr point to offset
    ConstantInt *Zero = ConstantInt::get(Type::getInt64Ty(F->getContext()), 0);
    // nglog: 在IR指令翻譯的開頭curr_data_offset預留的位置, 就是存放返回值
    Value *gepinst = irbuilder.CreateGEP(gv_data_seg, {Zero, Zero}, "");
    // cast gep_ptr to value->type
    PointerType *target_ptr_type = PointerType::get(F->getReturnType(), cast(gepinst->getType())->getAddressSpace());
    Value *ptr = irbuilder.CreatePointerCast(gepinst, target_ptr_type);
    // load return value
    Value *retval = irbuilder.CreateLoad(ptr);
 
    ReturnInst *inst_ret = ReturnInst::Create(this->Mod->getContext(), retval, ret_bbl);
}
else
{
    ReturnInst *inst_ret = ReturnInst::Create(this->Mod->getContext(), ret_bbl);
}
 
irbuilder.CreateBr(ret_bbl); 
```

5. 一些問題
-------

記錄一下我在嘗試把 xVMP 移值到 LLVM 19 時遇到的問題 (不一定是 xVMP 問題，也可以是我移植的問題)。

1.  clone 解釋器函數

上述複製`vm_interpreter()`的做法在 LLVM19 已不可行，會報錯：

```
Assertion `(NewFunc->getParent() == nullptr || NewFunc->getParent() == OldFunc->getParent()) && "Expected NewFunc to have the same parent, or no parent"' failed.

```

`CloneFunctionInto()`函數現在只允許`NewFunc`要麼與`OldFunc`具有相同的 parent，要麼`NewFunc`沒有 parent。

當時不太理解為什麼要 clone `vm_interpreter()`，以為只是為了解決命名沖突的問題，因此嘗試只調用`setName()`來修改函數名字。

```
FunctionCallee tmp = M->getOrInsertFunction(tempFunc.getName(), tempFunc.getFunctionType());
Function* newFunc = cast(tmp.getCallee());
newFunc->setLinkage(GlobalValue::LinkageTypes::InternalLinkage);
newFunc->setName(tempFunc.getName() + "_" + F->getName()); 
```

結果會報另一個錯：

```
Global is external, but doesn't have external or weak linkage!
ptr @_Z7startVMv__Z3addii
LLVM ERROR: Broken module found, compilation aborted!

```

意思是：「這個全域符號（GlobalVariable / Function / GlobalAlias）**在語義上是外部可見的（external），但是卻沒有給它外部連結屬性（ExternalLinkage / WeakLinkage / ExternalWeakLinkage …），這種組合是非法的**」

雖然可以將 Linkage 改成`ExternalLinkage`來解決上面這個問題，但在之後會鏈接錯誤：

```
/usr/bin/ld: /tmp/output-c589ca.o: in function `add(int, int)':
test.cpp:(.text+0x21): undefined reference to `_Z7startVMv__Z3addii'
/usr/bin/ld: /tmp/output-c589ca.o: in function `add2(int, int, int)':
test.cpp:(.text+0x6f): undefined reference to `_Z7startVMv__Z4add2iii'
clang++: error: linker command failed with exit code 1 (use -v to see invocation)

```

最終的解決方法是創建一個沒有 parent 的函數`NewFn`，用`Fn`來初始化`NewFn`，最後再 clone 並添加到當前 Module 中。

```
// 將 SrcMod 中的 Fn 克隆到 DstMod，返回已掛在 DstMod 的新函數
Function* VMInterpreter::cloneFunctionAcrossModules(Function &Fn, Module &DstMod, Twine NewName) {
    // 1. 建一個「無所屬」的空函數
    FunctionType *FTy = Fn.getFunctionType();
    Function *NewFn = Function::Create(FTy, Fn.getLinkage(),
                                       Fn.getName(), /*Parent=*/nullptr);
    NewFn->copyAttributesFrom(&Fn);
    NewFn->setComdat(Fn.getComdat());
    NewFn->setVisibility(Fn.getVisibility());
 
    NewFn->setName(NewName);
 
    if (Fn.isDeclaration()) {
        // 只是聲明，直接插進目標模組即可
        DstMod.getFunctionList().push_back(NewFn);
        return NewFn;
    }
 
    // 2. 填形參映射
    ValueToValueMapTy VMap;
    auto DI = NewFn->arg_begin();
    for (const Argument &AI : Fn.args()) {
        DI->setName(AI.getName());
        VMap[&AI] = &*DI++;
    }
 
    // 3. 真正克隆本體（此時 NewFn 還沒 Parent，符合 assertion）
    SmallVector Returns;
    CloneFunctionInto(NewFn, &Fn, VMap,
                      CloneFunctionChangeType::LocalChangesOnly,
                      Returns);
 
    // 4. 現在再掛到目標模組
    DstMod.getFunctionList().push_back(NewFn);
    return NewFn;
} 
```

1.  `alloca`指令的問題

xVMP 中對`alloca`指令的轉譯部份如下，它是在`gv_data_seg`預留了一片位置來模擬其分配的棧空間。

```
// nglog: 預留位置, 用於模擬AllocaInst分配的棧空間
// AllocaInst_alloca_area
int area_offset = curr_data_offset;
int alloca_size = modDataLayout->getTypeAllocSize(inst->getAllocatedType());
curr_data_offset += alloca_size;
 
// nglog: 構建AllocaInst的字節碼
std::vector hex_code;
ins_to_hex(hex_code, pack_op(ALLOCA_OP), packed_res, pack(area_offset, POINTER_SIZE)); 
```

一開始沒有看仔細，以為它的解釋器在解釋`alloca`指令時，`<result>`是以`data_off`來保存的。

後來回頭再看才發現，它保存的是一個真實的地址 (virtual address)，指向`data_seg_addr + area_offset`，而`data_seg_addr`就是`gv_data_seg`的基址。

```
void alloca_handler()
{
    // size and type of pointer is useless
    uint8_t var_size = get_byte_code();
    uint8_t var_type = get_byte_code();
 
    // get pointer var offset
    uint64_t var_offset = unpack_code(POINTER_SIZE);
 
    // get alloca area offset
    uint64_t area_offset = unpack_code(POINTER_SIZE);
 
    // store area virtual address to var
    // set_var(var_offset, POINTER_SIZE, data_seg_addr+area_offset);
    pack_store_addr(data_seg_addr + var_offset, data_seg_addr + area_offset, var_size);
}

```

假如只保存`area_offset`會有什麼問題？在不涉及指針操作時不會有太大問題。

而一旦遇到像`my_memcpy()`這樣操作指針的函數時，會出現有問題。

```
void my_memcpy(int* a, int *b) {
    *a = *b;
}
 
__attribute((__annotate__(("ngvmp"))))
int test5() {
    int a = 0;
    int b = 10;
    my_memcpy(&a, &b);
     
    return a;
}

```

`test5()`函數未經 vmp 前的 ir 代碼如下：

```
; Function Attrs: mustprogress noinline nounwind optnone uwtable
define dso_local noundef i32 @_Z5test5v() #4 {
entry:
  %a = alloca i32, align 4
  %b = alloca i32, align 4
  store i32 0, ptr %a, align 4
  store i32 10, ptr %b, align 4
  call void @_Z9my_memcpyPiS_(ptr noundef %a, ptr noundef %b)
  %0 = load i32, ptr %a, align 4
  ret i32 %0
}


```

而 vmp 後，`call`指令的 handler 如下。

![](https://bbs.kanxue.com/upload/attach/202510/946537_HNV24DCK5AU4ZWJ.webp)

問題在於`var_a`和`var_b`的值是`alloca`指令預留的`data_off`。

而`my_memcpy()`中會把`var_a`和`var_b`視為指針進行相關操作，即把`0xC`視為地址，這顯然會導致報錯。

![](https://bbs.kanxue.com/upload/attach/202510/946537_68ZGWJCS765A3QD.webp)

因此`alloca`的`<result>`必須被保存為一個真實的地址，這也是為什麼 xVMP 要把`gv_data_seg`轉換成地址後保存到`data_seg_addr`，後續的操作也是通過`data_seg_addr`來執行。

1.  間接函數調用問題

以下是一個 llvm ir 形式的間接函數調用。

根據 xVMP 的邏輯，在轉譯`store ptr @_Z15on_ace_detectedPKc, ptr %Func, align 8`時理應會把`@_Z15on_ace_detectedPKc`函數的實際地址保存到`%Func`。

```
; Function Attrs: mustprogress noinline optnone uwtable
define dso_local void @_Z5test2v() #0 {
entry:
  %Func = alloca ptr, align 8
  store ptr @_Z15on_ace_detectedPKc, ptr %Func, align 8
  %0 = load ptr, ptr %Func, align 8
  call void %0(ptr noundef @.str.2)
  ret void
}


```

問題是`@_Z15on_ace_detectedPKc`的類型是`GlobalValue*`，而`packValue()`中沒有針對`GlobalValue*`的處理，因此最終的`gv_value_map`中也不會有`@_Z15on_ace_detectedPKc`。

注：`GlobalVariable`繼承自`GlobalValue`。

```
// if value not in map
if (value_map->find(value) == value_map->end())
{
    // check value is not a GlobalVariable
    if (GlobalVariable *gv = dyn_cast(value))
    {
        // is a GlobalVariable and not in value_map
        // put it into value_map
        insert_to_value_map(value_map, value, curr_data_offset);
 
        // also put it into gv_value_map
        gv_value_map.insert(pair(gv, curr_data_offset));
 
        int res_size = modDataLayout->getTypeAllocSize(gv->getType());
        curr_data_offset += res_size;
    }
    else
    {
        assert(value_map->find(value) != value_map->end());
    }
} 
```

這會繼而導致之後在遍歷`gv_value_map`時忽略了對`@_Z15on_ace_detectedPKc`的處理 ( 這裡的處理指：獲取實際地址 → 寫到`gv_data_seg` )。

```
for (auto p : *gv_value_map)
{
    GlobalVariable *gv = p.first;
    int offset = p.second;
 
    errs() << "[*] storing GlobalVariable: " << *gv << "\t" << offset << "\n";
 
    // convert pointer to int64
    Value *gv_addr_int = irbuilder.CreatePtrToInt(gv, Type::getInt64Ty(Mod->getContext()));
 
    // create gep: data_seg + offset
    ConstantInt *Zero = ConstantInt::get(Type::getInt64Ty(Mod->getContext()), 0);
    Value *offset_value = ConstantInt::get(Type::getInt64Ty(Mod->getContext()), offset);
    Value *gepinst = irbuilder.CreateGEP(gv_data_seg, {Zero, offset_value}, "");
 
    // convert gep i8* to i64*
    PointerType *target_ptr_type = PointerType::get(gv_addr_int->getType(), cast(gepinst->getType())->getAddressSpace());
    Value *ptr = irbuilder.CreatePointerCast(gepinst, target_ptr_type);
 
    // store gv_addr_int to data_seg+offset
    irbuilder.CreateStore(gv_addr_int, ptr);
} 
```

最終 vm 在執行時，會調用一個空地址，導致段錯誤。

1.  遺留的全局變量

`call`指令的參數有可能是全局變量，但 xVMP 在處理時似乎忽略了這點。

```
void GOVMTranslator::handle_callinst(CallInst *inst, long long curr_func_id)
{
    // nglog: 指向new_call_handler的待插入點
    IRBuilder<> IRBcon(this->callinst_handler_conBBL);
 
    // firstly, we need to unpack function args
    std::vector target_func_args;
    for (unsigned idx = 0; idx < inst->getNumArgOperands(); idx++)
    {
        Value *currarg = inst->getArgOperand(idx);
         
        // nglog: 常量直接保存在target_func_args
        // if value is a constant, use it directly
        if (ConstantData *CD = dyn_cast(currarg))
        {
            target_func_args.push_back(currarg);
            continue;
        }
                // 假如currarg是全局變量, 則它不會在value_map中, 繼而這裡取的curroffset是錯的
        unsigned curroffset = value_map[currarg];
        // ...
} 
```

除此之外，xVMP 並不支援`phi`、`select`、`switch`、`extractvalue`、`insertvalue`等指令，不過這些指令適配起來也相對簡單。

但`invoke`指令似乎無法處理？這使得代碼中一旦有`std`相關的邏輯就會無法被 vmp。

( 各位靚仔如有處理`invoke`指令的思路還請不吝賜教！！！ )

6. 測試
-----

樣例：https://github.com/JonathanSalwan/Tigress_protection/blob/master/samples/sample10.c  
(如有其他更複雜的樣例也歡迎提供給我！)

vmp 後的效果大致如下。

![](https://bbs.kanxue.com/upload/attach/202510/946537_NA4MPJ6S9YNKWHQ.webp)

配合控制流平坦化，可達到以下效果。

![](https://bbs.kanxue.com/upload/attach/202510/946537_KMRC66WCHR6NDAS.webp)

[[培训] 传播安全知识、拓宽行业人脉——看雪讲师团队等你加入！](https://bbs.kanxue.com/thread-275828.htm)

[#基础知识](forum-41-1-130.htm) [#虚拟化](forum-41-1-136.htm)