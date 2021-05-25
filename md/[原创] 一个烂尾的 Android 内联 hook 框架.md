> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-267739.htm)

> [原创] 一个烂尾的 Android 内联 hook 框架

1. 引言
-----

  之前因为有某些需求想找一个 arm 平台的 inline hook 框架，但是没能找到一个既精简、又支持修改多参数、修改返回值、支持用自定义函数替换掉原函数的框架。所以就决定花点时间自己实现一个，写了几天以后被告知不用做了，于是便有了这个烂尾项目。今天整理文件的时候看到了，与其留着发霉不如分享给坛友作为学习的例子。此代码为没用的草稿，不合规范的地方就不用对其提意见了。

2. 需求分析与初步设计
------------

  2.1 精简  
    使用一个结构体和一段 shellcode 维护一个 hook  
  2.2 支持多线程 hook  
    每个 hook 维护一段只属于自己的 shellcode  
  2.3 支持修改参数  
    参数数量小于 4 的修改 r0-r3 寄存器，大于 4 的再修改栈空间  
  2.4 支持修改返回值  
    覆盖 r0 寄存器的值  
  2.5 支持函数替换  
    hook 函数执行完以后加载原函数下一条指令地址到 lr 寄存器  
    总体设计为，每个 hook 维持一个结构体与一段自己的 shellcode，初始化 hook 时直接修改 shellcode 中的变量值，备份并修改原函数的起始汇编代码，使其直接跳到 shellcode 执行，完成其余的工作。所有的 hook 保存在一个链表中进行统一管理。

3. 具体实现
-------

    以下为维护 hook 的结构体：

```
typedef struct inline_item{
    void* shell_code;           // shelcode的内存起始地址
    int32_t shell_code_len;     // shellcode的长度
    address_t old_fun;          // 原始函数的起始地址
    address_t new_fun;          // hook函数的起始地址
    void* back_code;            // 备份的代码
    int32_t back_code_len;      // 备份代码的长度
    void* reg;                  // 保存寄存器空间
    void* stack;                // 保存栈空间
    list_node_t* param_list;    // 参数链表
    void* ret_value;            // 返回值
} inline_item_t;

```

    在初始化 hook 时，首先获取 shellcode 中变量的内存地址，分配一块内存空间给结构体中的 shellcode_code 字段，并记录长度，记录原始函数的地址与 hook 函数的地址，存储原函数的前几条指令作为备份代码，并记录备份代码的长度，param_list 中存储有想要修改的参数值，ret_value 中存储有想要修改成的返回值。

 

    初始化 hook 的代码如下：

```
inline_item_t* init_item(address_t* target_fun, address_t* hook_fun, list_node_t* param_list,address_t ret_value)
{
    inline_item_t* p_item = 0;
    address_t temp = 0;
    p_item = (inline_item_t*)malloc( sizeof(inline_item_t) );
    p_item->shell_code_len = (int8_t*)&asm_shellcode_end - (int8_t*)&asm_shellcode_begin;
    p_item->shell_code = (int8_t*)malloc( p_item->shell_code_len);  // 分配shellcode空间
    p_item->old_fun = target_fun;                                   // 保存原始函数地址
    p_item->new_fun = hook_fun;                                     // 存储hook函数地址
    p_item->back_code_len = 12;                                     // 备份12字节指令
    p_item->back_code = malloc(p_item->back_code_len);
    p_item->reg = malloc(0x10 * sizeof(address_t));                 // 分配备份寄存器的空间
    p_item->stack = malloc(0x10 * sizeof(address_t));               // 分配备份栈的空间
    p_item->param_list = 0;
    p_item->ret_value = 0;
    if( param_list != 0){
        p_item->param_list = param_list;                            // 初始化要修改的参数列表
    }
    if( ret_value!=0 ){
        p_item->ret_value = (address_t*)malloc( sizeof(address_t) ); // 初始化要修改的返回值
        ret_value = 10;
        memcpy( p_item->ret_value, &ret_value, sizeof(address_t));
    }
    return p_item;
}

```

    在备份原函数前几条指令时需要对指令类型进行判断，如果是 thumb 指令，则需要对地址减 1 再进行保存。  
    判断指令类型的代码如下：

```
bool get_type(address_t addr)  
{
#if defined(__arm64__) || defined(__aarch64__)
    return ((int64_t)addr & 0x1);
#else
    return ((int32_t)addr & 0x1);  
#endif
}

```

    备份指令的函数如下：

```
void back_target(inline_item_t* p_item){
    p_item->back_code_len = BACKLEN;
    //保存old_fun前几条指令
    p_item->back_code = (int8_t*)malloc(p_item->back_code_len);
    memcpy(p_item->back_code, (address_t*)p_item->old_fun, p_item->back_code_len);
    memcpy(p_item->shell_code + asm_pos.back_code_pos, p_item->back_code, p_item->back_code_len);
    //修改old_fun前几条指令
    if( !set_mem_permission(p_item->old_fun,RWX) ){ // 修改内存属性为可读可写可执行
        return ;
    }
    memcpy((address_t*)p_item->old_fun, p_item->shell_code + asm_pos.load_pc_pos,sizeof(address_t));
    memcpy((address_t*)p_item->old_fun + 1, &p_item->shell_code, sizeof(address_t));
    memcpy(p_item->shell_code+asm_pos.load_pc_pos, p_item->back_code, sizeof(address_t)*3);
}

```

    指令地址为奇数是 thumb 指令  
    在 shellcode 开始执行之前，需要先对其中的变量进行初始化，以下为需要初始化的值：

```
asm_old_fun_continue:   // 原函数的地址
    .word 0x12345678   
asm_new_fun:            // hook函数的地址           
    .word 0x12345678   
asm_param_num:          //参数的个数
    .word 0x00000010
asm_reg_sp:             // 栈顶值
    .word 0x12345678
asm_reg_lr:             // 返回地址
    .word 0x12345678
callback_modify_param:  // callback_modify_param函数的地址
    .word 0x00000000
callback_modify_return: // callback_modify_return函数的地址
    .word 0x00000000
asm_item_addr:          // 当前hook结构体地址
    .word 0x12345678
asm_reg_addr:         
    .word 0x12345678
asm_stack_addr:
    .word 0x12345678
 
asm_back_code:          // 备份的原始函数代码块
    .word 0x12345678
    .word 0x12345678
    .word 0x12345678
    ldr r15,asm_old_fun_continue    // 跳转到备份代码的下一条指令继续执行

```

    shellcode 在执行时，需要先保存原始上下文，由于 r13,r14,r15 寄存器比较特殊，在跳转到保存上下文代码块时，寄存器的值已经发生变化，因此需要单独对其进行保存

```
str r13,asm_reg_sp
str r14,asm_reg_lr
// stmfd sp!,{r0}       // 这个需要注意，保存的pc应当是old_fun的下一条指令地址，
// sub r0,r15,0x        // 所以执行到这里以后要保存当前pc减去已经执行的指令步数*4
// ldmfd sp!,{r0}
bl fun_save_context

```

    再进行上下文环境保存，代码如下：

```
fun_save_context:
    stmfd sp!,{r12} // ------- 开始保存寄存器
    ldr r12,asm_reg_addr
    stmea r12!,{r0-r11}      //保存r0--r11
    mov r0,r12
    ldmfd sp!,{r12}          //取出r12
    stmea r0!,{r12}          //保存r12
    mrs r1,cpsr              //保存CPSR寄存器
    stmea r0!,{r1}
   // mov r1,sp                //不能直接操作sp
   // stmea r0!,{r1} // ------- 寄存器保存完成
    ldr r0,asm_stack_addr //-- 开始保存栈空间
    mov r2,0x10              //需要保存的栈空间大小
    mov r1,sp
 _loop_save_stack:
    cmp r2,0x00
    ble _loop_save_context_end
    ldr r3,[r1]
    str r3,[r0],0x04
    add r1,r1,0x04
    sub r2,r2,0x01
    b _loop_save_stack
 _loop_save_context_end:  //-- 栈空间保存完成
    bx r14
fun_save_context_end:

```

    与保存上下文相对应的，以下是恢复上下文的代码块：

```
fun_back_context:
    ldr r0,asm_stack_addr   // 开始恢复栈空间
    mov r1,sp
    mov r2,0x10
 _loop_back_stack:
    cmp r2,0x00
    ble _loop_back_stack_end
    ldr r3,[r0]
    str r3,[r1],0x04
    add r0,r0,0x04
    sub r2,r2,0x01
    b _loop_back_stack
 _loop_back_stack_end:      // 恢复栈空间完成
    ldr r12,asm_reg_addr
    ldmfd r12!,{r0-r11}     // 先取出r0-r11
    stmfd sp!,{r0-r1}
    mov r0,r12
    ldmfd r0!,{r12}         // 取出r12
    ldmfd r0!,{r1}
    msr cpsr,r1
   // ldmfd r0!,{r1}
   // mov sp,r1
    ldmfd sp!,{r0-r1}
    bx r14
fun_back_context_end:

```

    执行 hook 的代码：

```
    ldr r13,asm_reg_sp
 
    bl fun_back_context     // 保存上下文
    ldr r0,asm_new_fun
    blx r0                  // 带返回跳转到hook函数
 
    stmfd sp!,{r0,r1}     
    ldr r1,callback_modify_param
    cmp r1,0x00
    beq not_modofy_param    // 判断是否需要修改参数
    ldr r0,asm_item_addr
    blx r1                  // 如果需要修改参数，以hook结构体作为参数带返回跳转callback_modify_param
not_modofy_param:
    ldmfd sp!,{r0,r1}
 
    ldr r13,asm_reg_sp
    bl fun_back_context
    bl asm_back_code         // 执行原函数
 
    stmfd sp!,{r0,r1}
    ldr r1,callback_modify_return
    cmp r1,0x00
    beq not_modofy_return    // 执行完原函数后，判断是否需要修改返回值
    ldr r0,asm_item_addr     // 如果需要修改返回值，以hook结构体作为参数带返回跳转
    blx r1
not_modofy_return:
    ldmfd sp!,{r0,r1}
 
    bl fun_back_context     // 恢复上下文
    ldr r14,asm_reg_lr      // 将old_fun的lr地址填充回去
    bx r14                  // 跳转回正常执行流程

```

    以上代码中，callback_modify_param 被填充为以下函数：

```
void modify_param(inline_item_t* p_item ){
    int n;
    int index;
    address_t data;
    list_node_t* param_list ;
    if(!p_item){
        return ;
    }
    param_list = p_item->param_list;  // 参数存储在结构体的一个链表中
    while(param_list != 0){           // 遍历整个链表
        index = ((param_t*)param_list->data)->index;
        data = ((param_t*)param_list->data)->data;
        if( index <=3 ){    // 如果参数小于3个，修改寄存器的值
            memcpy(p_item->reg + sizeof(address_t)*index, &data, sizeof(address_t) );
        }
        else {              // 参数大于3个，修改栈顶值
            memcpy(p_item->stack + sizeof(address_t)*(index-4), &data, sizeof(address_t));
        }
        param_list = param_list->next;
    }
}

```

    callback_modify_return 的值被填充为以下函数的地址：

```
void modify_return(inline_item_t*p_item){
    if(!p_item)    {
        return ;
    }
    //覆盖r0的值修改返回值
    memcpy(p_item->reg, p_item->ret_value, sizeof(address_t));
}

```

    指令修复部分还没完成。

 

    最后附上 Makefile:

```
NDK_ROOT=~/Android/Sdk/ndk/21.0.6113669
TOOL_CHAIN=$(NDK_ROOT)/lone_toolchain_23
CC=$(TOOL_CHAIN)/bin/armv7a-linux-androideabi23-clang
CFLAG=-g -m32
TARGET=sniper
RM=rm -f
DIR=.
OBJ_DIR=$(DIR)/obj
SRC_DIR=$(DIR)/src
SRC=$(wildcard $(DIR)/*.c)
SRC+=$(wildcard $(DIR)/*.asm)
OBJ=$(patsubst %.asm,%.o,$(patsubst %.c,%.o,$(SRC)))
all:$(TARGET)
$(TARGET):$(OBJ)
    $(CC) $(CFLAG) -o $@ $^
$(DIR)/%.o:$(DIR)/%.c
    $(CC) $(CFLAG) -c $<
$(DIR)/%.o:$(DIR)/%.asm
    $(CC) $(CFLAG) -c $<
clean:
    $(RM) *.o
    $(RM) *.s
install:
    adb push $(TARGET) /sdcard

```

4. 结语
-----

    总体来讲 arm 上的 inline hook 框架要比 x86 难实现一点，传参方式使用寄存器加栈顶空间，还需要对两种不同类型的指令进行判断和修复。目前的代码只能说是写了一个开头，要想做一个完整的框架，还是需要一点工作量的。

[[看雪官方培训] Unicorn Trace 还原 Ollvm 算法！《安卓高级研修班》2021 年 6 月班火热招生！！](https://bbs.pediy.com/thread-267018.htm)

最后于 1 天前 被某警官编辑 ，原因：