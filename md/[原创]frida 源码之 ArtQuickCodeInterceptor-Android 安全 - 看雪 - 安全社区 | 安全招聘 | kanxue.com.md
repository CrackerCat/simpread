> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-287333.htm)

> [原创]frida 源码之 ArtQuickCodeInterceptor

```
class ArtQuickCodeInterceptor {
  constructor (quickCode) {
    this.quickCode = quickCode; //原函数入口地址
    this.quickCodeAddress = (Process.arch === 'arm') //arm架构下的地址
      ? quickCode.and(THUMB_BIT_REMOVAL_MASK)
      : quickCode;
 
    this.redirectSize = 0; //要跳转的代码大小
    this.trampoline = null;  //hook用的跳板地址
    this.overwrittenPrologue = null; //被覆盖的原始指令保存区
    this.overwrittenPrologueLength = 0; //被覆盖的字节长度
  }
  // 判断能否安全的复制原函数前relocationSize字节到trampoline
  _canRelocateCode (relocationSize, constraints) {
    const Writer = thunkWriters[Process.arch]; //构造新的跳板代码
    const Relocator = thunkRelocators[Process.arch];//读取原始的代码尝试分析、调整
 
    const { quickCodeAddress } = this;
 
    const writer = new Writer(quickCodeAddress);
    const relocator = new Relocator(quickCodeAddress, writer);
 
    let offset;
    if (Process.arch === 'arm64') {
      // 为hook插入跳板 用到临时寄存器 x16 x17
      // 分析原函数前部指令时 检查它有没有使用这些寄存器
      let availableScratchRegs = new Set(['x16', 'x17']);
 
      do {
        const nextOffset = relocator.readOne();
 
        const nextScratchRegs = new Set(availableScratchRegs);
        //寄存器逻辑检测 获取当前指令读取和写入的寄存器
        const { read, written } = relocator.input.regsAccessed;
        for (const regs of [read, written]) {
          //将使用的寄存器统一转换为x16风格 可以从集里剔除
          for (const reg of regs) {
            let name;
            if (reg.startsWith('w')) {
              name = 'x' + reg.substring(1);
            } else {
              name = reg;
            }
            nextScratchRegs.delete(name);
          }
        }
        if (nextScratchRegs.size === 0) {
          break;
        }
 
        offset = nextOffset;
        availableScratchRegs = nextScratchRegs;
      } while (offset < relocationSize && !relocator.eoi);
 
      constraints.availableScratchRegs = availableScratchRegs;
    } else {
      do {
        //指令扫描  读取指令并尝试relocate 直到达到relocationSize字节 或达函数末尾
        offset = relocator.readOne();
      } while (offset < relocationSize && !relocator.eoi);
    }
    //最终判断 能成功relocate足够多的指令 就返回true 就可以安全插入跳板
    return offset >= relocationSize;
  }
  //为一个需要 Hook 的 native 方法分配 trampoline（跳板）代码区域
  
  _allocateTrampoline () {
    //首次调用时 创建一个 trampoline allocator  分配可执行内存块
    if (trampolineAllocator === null) {
      const trampolineSize = (pointerSize === 4) ? 128 : 256;
      trampolineAllocator = makeCodeAllocator(trampolineSize);
    }
    //获取当前架构下最大的hook跳转指令覆盖范围
    const maxRedirectSize = artQuickCodeHookRedirectSize[Process.arch];
    /*
    redirectSize 覆盖原函数前多少字节
    spec 分配跳板的要求 地址对齐 是否靠近目标函数
    alignment 分配对齐 默认1 arm64强制4096
    constraints 记录像arm64可用的scratch寄存器限制
    */
    
    let redirectSize, spec;
    let alignment = 1;
    const constraints = {};
    //判断是否可以安全relocate原函数指令
    //如果是32位 直接maxRedirectSize 否则检查原函数前导指令能否被relocate （_canRelocateCode）
    //如果成功可以patch 使用默认的分配要求
    if (pointerSize === 4 || this._canRelocateCode(maxRedirectSize, constraints)) {
      redirectSize = maxRedirectSize;
 
      spec = {};
    } else {
      //如果不能原地patch 使用近跳 near jump 跳板
 
      let maxDistance;
      if (Process.arch === 'x64') {
        redirectSize = 5;
        maxDistance = X86_JMP_MAX_DISTANCE;
      } else if (Process.arch === 'arm64') {
        //ADRP + ADD跳板8字节
        //跳转距离受ADRP限制 +- 4GB
        //需要跳板按4kb对齐
        redirectSize = 8;
        maxDistance = ARM64_ADRP_MAX_DISTANCE;
        alignment = 4096;
      }
 
      spec = { near: this.quickCodeAddress, maxDistance };
    }
 
    this.redirectSize = redirectSize;
    //分配跳板内存 通常是executable区段分配
    this.trampoline = trampolineAllocator.allocateSlice(spec, alignment);
    //返回寄存器使用限制信息
    return constraints;
  }
 
  _destroyTrampoline () {
    trampolineAllocator.freeSlice(this.trampoline);
  }
  /*
  在内存中写 trampoline（跳板逻辑）
  备份原函数前导指令
  用跳转指令覆盖原始函数前导部分，使其跳向 trampoline
  */
  activate (vm) {
    //分配 trampoline 并检查可跳转条件
    //检查是否能在原函数前面插入跳板 并分配跳板的内存
    const constraints = this._allocateTrampoline();
    //获取跳板相关参数
    /*
    trampoline 已分配跳板的地址
    quickCode 原方法的入口地址
    redirectSize 将要覆盖的原函数字节长度
    */
    const { trampoline, quickCode, redirectSize } = this;
    // 写跳板内容 即跳板函数
    /*
    把原函数的前导指令复制到跳板函数中
    添加hook逻辑
    再跳回原函数
    返回prologueLength 跳板函数中复制了多少字节原始函数 用于后续还原
    */
 
 
    const writeTrampoline = artQuickCodeReplacementTrampolineWriters[Process.arch];
    const prologueLength = writeTrampoline(trampoline, quickCode, redirectSize, constraints, vm);
    //备份原函数前导指令
    // 把原函数开头被覆盖的部分保存起来 
    this.overwrittenPrologueLength = prologueLength;
    // 重写原函数开头 跳到跳板函数
    
    this.overwrittenPrologue = Memory.dup(this.quickCodeAddress, prologueLength);
    //调用writePrologue函数 把原函数开头redirectSize字节覆盖为跳转指令
    const writePrologue = artQuickCodePrologueWriters[Process.arch];
    writePrologue(quickCode, trampoline, redirectSize);
  }
  /*
  撤销一个函数 hook，恢复原始的指令代码，并清理 trampoline
 
  读取之前保存的原始代码（函数前 prologue）
  写回目标函数地址处
  销毁跳板代码，完成 hook 卸载
  */
  deactivate () {
    //quickCodeAddress 被hook的目标函数地址
    //overwrittenPrologueLength 原本被覆盖的字节数
    const { quickCodeAddress, overwrittenPrologueLength: prologueLength } = this;
    //选择合适的架构
    const Writer = thunkWriters[Process.arch];
    //传入要修改的起始地址和修改的长度 quickCodeAddress prologueLength
    Memory.patchCode(quickCodeAddress, prologueLength, code => {
      //设置程序计数器pc为quickCodeAddress 方便写入相当于当前位置的指令
      const writer = new Writer(code, { pc: quickCodeAddress });
      //从this中提取保存的原始函数开头内容 
      const { overwrittenPrologue } = this;
      //将保存的原始字节读出来 重新写入目标地址 恢复原状
      writer.putBytes(overwrittenPrologue.readByteArray(prologueLength));
      //刷新写入器缓冲区 保证指令写入生效
      writer.flush();
    });
    //调用私有方法 销毁hook创建的跳板代码
    this._destroyTrampoline();
  }
}
```

![](https://bbs.kanxue.com/upload/attach/202506/1030887_PRCA363YPWF5MGW.webp)

[[培训] 科锐逆向工程师培训第 53 期 2025 年 7 月 8 日开班！](https://bbs.kanxue.com/thread-51839.htm)

最后于 23 分钟前 被 mb_erlnpqnh 编辑 ，原因：

[#基础理论](forum-161-1-117.htm) [#HOOK 注入](forum-161-1-125.htm) [#源码框架](forum-161-1-127.htm)