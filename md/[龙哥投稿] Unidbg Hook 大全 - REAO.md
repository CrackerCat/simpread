> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [reao.io](https://reao.io/archives/90/)

> [scode type="green"]龙哥投稿，这里推荐一下龙哥的星球，干货多多 [/scode] 文章采用 Frida + Unidbg 代码对照的方式，帮助熟悉 Frida 的读者尽快理解 Unid...

Loading...

龙哥投稿，这里推荐一下龙哥的星球，干货多多

[![](https://reao.io/usr/uploads/2021/10/3605859095.png)](https://reao.io/usr/uploads/2021/10/3605859095.png)

[image.png](https://reao.io/usr/uploads/2021/10/3605859095.png)

* * *

文章采用 Frida + Unidbg 代码对照的方式，帮助熟悉 Frida 的读者尽快理解 Unidbg Hook。

样例是自写 DEMO，可百度云下载源码以及 APK。

链接：[https://pan.baidu.com/s/18WQnWeSvdcrwCZ_q-LobQA](https://pan.baidu.com/s/18WQnWeSvdcrwCZ_q-LobQA)  
提取码：6666

### 一、基础知识

#### 获取指定 SO 文件的基地址

frida version

```
function hook_module() {
    var baseAddr = Module.findBaseAddress("libnative-lib.so");
    console.log("baseAddr", baseAddr);
}


```

Unidbg 中如下

```
// 加载so到虚拟内存
DalvikModule dm = vm.loadLibrary("libnative-lib.so", true);
// 加载好的so对应为一个模块
module = dm.getModule();
// 打印libnative-lib.so在Unidbg虚拟内存中的基地址
System.out.println("baseAddr:"+module.base);


```

加载了多个 SO 的情况

```
// 获取某个具体SO的句柄
Module yourModule = emulator.getMemory().findModule("yourModuleName");
// 打印其基地址
System.out.println("baseAddr:"+yourModule.base);


```

如果只主动加载一个 SO，其基址恒为 0x40000000 , 这是一个检测 Unidbg 的点，可以在 `com/github/unidbg/memory/Memory.java` 中做修改

```
public interface Memory extends IO, Loader, StackMemory {

    long STACK_BASE = 0xc0000000L;
    int STACK_SIZE_OF_PAGE = 256; // 1024k

	// 修改内存映射的起始地址
    long MMAP_BASE = 0x40000000L;

    UnidbgPointer allocateStack(int size);
    UnidbgPointer pointer(long address);
    void setStackPoint(long sp);


```

获取 SO 中导出函数的地址

```
// 加载so到虚拟内存
DalvikModule dm = vm.loadLibrary("libnative-lib.so", true);
// 加载好的 libscmain.so对应为一个模块
module = dm.getModule();
int address = (int) module.findSymbolByName("funcNmae").getAddress();


```

非导出函数，需要在 IDA 中确定偏移地址，`base`+`offset`获得真实地址

```
// 加载so到虚拟内存
DalvikModule dm = vm.loadLibrary("libnative-lib.so", true);
// 加载好的so对应为一个模块
module = dm.getModule();
// offset，在IDA中查看
int offset = 0x1234;
// 真实地址 = baseAddr + offset
int address = (int) (module.base + offset);


```

### 二、Hook 函数

Unidbg 模拟执行 DEMO 的代码如下

```
package com.hookInUnidbg;

import com.github.unidbg.AndroidEmulator;
import com.github.unidbg.Module;
import com.github.unidbg.linux.android.AndroidEmulatorBuilder;
import com.github.unidbg.linux.android.AndroidResolver;
import com.github.unidbg.linux.android.dvm.*;
import com.github.unidbg.memory.Memory;

import java.io.File;

public class crackDemo {
    private final AndroidEmulator emulator;
    private final VM vm;
    private final Module module;

    crackDemo() {

        // 创建模拟器实例
        emulator = AndroidEmulatorBuilder.for32Bit().build();

        // 模拟器的内存操作接口
        final Memory memory = emulator.getMemory();
        // 设置系统类库解析
        memory.setLibraryResolver(new AndroidResolver(23));
        // 创建Android虚拟机
        vm = emulator.createDalvikVM(new File("unidbg-android/src/test/resources/hookInUnidbg/app-debug.apk"));


        // 加载so到虚拟内存
        DalvikModule dm = vm.loadLibrary("hookinunidbg", true);
        // 加载好的 libhookinunidbg.so对应为一个模块
        module = dm.getModule();

        // 执行JNIOnLoad（如果有的话）
        dm.callJNI_OnLoad(emulator);
    }

    public void call(){
        DvmClass dvmClass = vm.resolveClass("com/example/hookinunidbg/MainActivity");
        String methodSign = "call()V";
        DvmObject<?> dvmObject = dvmClass.newObject(null);

        dvmObject.callJniMethodObject(emulator, methodSign);
    }


    public static void main(String[] args) {
        crackDemo mydemo = new crackDemo();
        mydemo.call();
    }
}


```

call 的流程中执行了数个函数，首先在 IDA 中观察 base64_encode 函数

```
int __fastcall base64_encode(int a1, unsigned int a2, int a3)
{
  int v3; // r2
  int v4; // r2
  int v5; // r2
  int v6; // r1
  int v7; // r1
  unsigned __int8 v9; // [sp+12h] [bp-26h]
  unsigned __int8 v10; // [sp+13h] [bp-25h]
  unsigned int i; // [sp+14h] [bp-24h]
  int v12; // [sp+18h] [bp-20h]
  int v14; // [sp+28h] [bp-10h]

  v12 = 0;
  v9 = 0;
  v14 = 0;
  for ( i = 0; i < a2; ++i )
  {
    v10 = *(_BYTE *)(a1 + i);
    if ( v12 )
    {
      if ( v12 == 1 )
      {
        v12 = 2;
        v4 = v14++;
        *(_BYTE *)(a3 + v4) = byte_273D[(16 * (v9 & 3)) | (v10 >> 4)];
      }
      else
      {
        v12 = 0;
        *(_BYTE *)(a3 + v14) = byte_273D[(4 * (v9 & 0xF)) | (v10 >> 6)];
        v5 = v14 + 1;
        v14 += 2;
        *(_BYTE *)(a3 + v5) = byte_273D[v10 & 0x3F];
      }
    }
    else
    {
      v12 = 1;
      v3 = v14++;
      *(_BYTE *)(a3 + v3) = byte_273D[v10 >> 2];
    }
    v9 = v10;
  }
  if ( v12 == 1 )
  {
    *(_BYTE *)(a3 + v14) = byte_273D[16 * (v9 & 3)];
    *(_BYTE *)(a3 + v14 + 1) = 61;
    v6 = v14 + 2;
    v14 += 3;
    *(_BYTE *)(a3 + v6) = 61;
  }
  else if ( v12 == 2 )
  {
    *(_BYTE *)(a3 + v14) = byte_273D[4 * (v9 & 0xF)];
    v7 = v14 + 1;
    v14 += 2;
    *(_BYTE *)(a3 + v7) = 61;
  }
  *(_BYTE *)(a3 + v14) = 0;
  return v14;
}


```

为了聚焦在 Hook 上，直接解释入参：参数 1 是传入的无符号字节数组，参数 2 是数组长度，参数 3 是一个 buffer，用于存放函数计算后的值，是一个字符串。我们打印入参和 buffer 的计算结果，从而确认函数是否是标准 base64。  
下面是 Frida 的版本，相信大家都不陌生。

```
// Frida Version
function main(){
    // get base address of target so;
    var base_addr = Module.findBaseAddress("libhookinunidbg.so");

    if (base_addr){
        var func_addr = Module.findExportByName("libhookinunidbg.so", "base64_encode");
        console.log("hook base64_encode function")
        Interceptor.attach(func_addr,{
            // 打印入参
            onEnter: function (args) {
                console.log("\n input:")
                this.buffer = args[2];
                var length = args[1];
                console.log(hexdump(args[0],{length: length.toUInt32()}))
                console.log("\n")
            },
            // 打印返回值
            onLeave: function () {
                console.log(" output:")
                console.log(this.buffer.readCString());
            }
        })
    }


}

setImmediate(main);


```

打印结果

```
[MIX 2S::hookInUnidbg]->
input:
           0  1  2  3  4  5  6  7  8  9  A  B  C  D  E  F  0123456789ABCDEF
cd1e72e0  6c 69 6c 61 63                                   lilac


output:
bGlsYWM=


```

接下来在 Unidbg 中做同样的事——打印入参，以及函数结束时打印 buffer。

首先从整体的角度来看一下 Unidbg 的 Hook 的体系，在 Android 方面可以分为两大类 [^1]

*   Unidbg 内置的 Hook 框架，包括 xHook/Whale/HookZz
*   基于 Unicorn Hook 的方案，主要指 Unicorn 原生 Hook 和 Unidbg 封装的 Console Debugger

**详细而言，一是 Unidbg 支持并内置的第三方 Hook 框架，有 Dobby(前身 HookZz)/Whale 这样的 Inline Hook 框架，也有 xHook 这样的 PLT Hook 框架。二是当选择 Unidbg 的底层引擎为 Unicorn 时，Unicorn 带来的 Hook 功能，Unicorn 提供了指令级和基本块粒度的 Hook，十分方便好用，而且 Unidbg 基于它封装了简单但足够强大的 Console Debugger**

我们该怎么选择 Hook 方案？这是一个问题。我的习惯是 _Console Debugger First！而且能用原生 Hook 就不用第三方_。  
为什么我会养成这样的习惯？文末我会给出自己的理由， 下面逐一讨论这些方案并实现等效 Frida 脚本功能的 Hook，也欢迎你和我讨论自己的观点。

首先，遇到一个可疑函数时，我们希望快速验证参数的内容，以及每个参数的含义，这里的关键在于快速。要知道，在一个复杂的样本里，会遇到几十甚至上百个看着不顺眼的函数，快速的 Hook 验证这些函数是否真有问题会大大提高分析效率。这时候我会用 Unidbg 的 Console Debugger，它基于 Unicorn Hook 封装和实现，Console Debugger 最适合_快速打击、快速验证_ 的场景，在这个场景上，它比 Unidbg 支持的其他任意方案都更敏捷，也比一台真实的 Android 测试机支持的任意 Hook 方案更快，甭管是 Frida 还是其他方案，都不会有它快。来看一下怎么驾驭它用，如下所示，使用 eulator.attach().addBreakPoint API 下断点，函数执行到此处会进入 Console Debugger 交互调试，值得一提的是，我选择在 JNI_OnLoad 前下断点，因为这是一个较早的时机点，可以避免错过潜在的、发生在 JNIOnLoad 中的函数调用。

```
// debug
emulator.attach().addBreakPoint(module.findSymbolByName("base64_encode").getAddress());

// 执行JNIOnLoad（如果有的话）
dm.callJNI_OnLoad(emulator);


```

必须要认识到，Console Debugger 或者其他基于 Unicorn Hook 的 Hook 方案，其都不具有 “函数” 的概念，我们以上面的 Frida 函数举例。onEnter 代表函数执行前的时机，我们一般在这里打印入参，onLeave 代表函数执行完的时机，在这里打印返回值，args 代表入参，return 代表返回值。但 Console Debugger 里则不然，addBreakPoint 等效于在 GDB 或者 IDA 调试中在某个地址下断点，代表的是地址处的这条汇编指令执行前的时机。因为像 IDA 调试中一样，需要使用者从指令的角度理解函数。

> 根据 ARM ATPCS 调用约定，当参数个数小于等于 4 个的时候，子程序间通过 R0~R3 来传递参数（R0-R3 代表参数 1 - 参数 4），如果参数个数大于 4 个，余下的参数通过 sp 所指向的数据栈进行参数传递。而函数的返回值通过 R0 传递回来。

比如此处，8A0 函数的入参有三个，所以由函数的调用方在调用 sub_8A0 前把参数 1、2、3 依次放在 R0-R2 中。

[![](https://reao.io/usr/uploads/2021/10/3873202867.png)](https://reao.io/usr/uploads/2021/10/3873202867.png)

[1.png](https://reao.io/usr/uploads/2021/10/3873202867.png)

r1 即参数 2，为 5，接下来查看 r0 和 r2 即参数 1 和参数 3，这两个参数看起来都是地址。为什么会把这两个 int 理解成地址呢？入参也可以是单纯数字嘛，这是因为 Unidbg 的内存映射从 0x40000000 开始，所以 400 开头的 int，多半是地址咯。

[![](https://reao.io/usr/uploads/2021/10/4270168143.png)](https://reao.io/usr/uploads/2021/10/4270168143.png)

[2.png](https://reao.io/usr/uploads/2021/10/4270168143.png)

mxx 功能等价于 Frida 中的 hexdump(xx)，数据展示上稍有不同，主要体现在两方面

*   frida hexdump 的展示上，左侧基地址从当前地址开始，而 Unidbg 从 0 开始，这点上 Frida 更好用
*   Unidbg 给出了所打印数据块的 md5 值，方便对比两块不同数据的内容是否一致，而且 Unidbg 展示数据的 Hex String，方便搜索

Console debugger 的交互命令类似 GDB，我们可以发现，m 也可以跟着地址，展示 hexdump(address)，但必须是十六进制且以 0x 开头。如下是 Console Debugger 支持的所有命令。需要再次强调，Console Debugger 基于 Unicorn 引擎，这意味着当使用 Dynarmic 引擎时，无法使用 Console Debugger。（Unidbg 支持多个底层引擎，在 Android 模拟执行上，最多使用的是 Unicorn 和 Dynarmic，Unicorn 提供的功能多，Unidbg 基于它封装了 Console Debugger 和诸多功能，而 Dynarmic 速度快，对比 Unicorn 快数倍甚至数十倍，更适合模拟执行的生产环境使用）

```
c: continue
n: step over
bt: back trace

st hex: search stack
shw hex: search writable heap
shr hex: search readable heap
shx hex: search executable heap

nb: break at next block
s|si: step into
s[decimal]: execute specified amount instruction
s(blx): execute util BLX mnemonic, low performance

m(op) [size]: show memory, default size is 0x70, size may hex or decimal
mr0-mr7, mfp, mip, msp [size]: show memory of specified register
m(address) [size]: show memory of specified address, address must start with 0x

wr0-wr7, wfp, wip, wsp <value>: write specified register
wb(address), ws(address), wi(address) <value>: write (byte, short, integer) memory of specified address, address must start with 0x
wx(address) <hex>: write bytes to memory at specified address, address must start with 0x

b(address): add temporarily breakpoint, address must start with 0x, can be module offset
b: add breakpoint of register PC
r: remove breakpoint of register PC
blr: add temporarily breakpoint of register LR

p (assembly): patch assembly at PC address
where: show java stack trace

trace [begin end]: Set trace instructions
traceRead [begin end]: Set trace memory read
traceWrite [begin end]: Set trace memory write
vm: view loaded modules
vbs: view breakpoints
d|dis: show disassemble
d(0x): show disassemble at specify address
stop: stop emulation
run [arg]: run test
cc size: convert asm from 0x400008a0 - 0x400008a0 + size bytes to c function


```

此处，我们要呈现 **打印参数 1 指向的内存块，长度为参数 2** 这样的效果，在 Frida 中 可以用 `console.log(hexdump(args[0],{length: args[1].toUInt32()}))`，而在 Console Debugger 中，暂不支持 `mr0 r1` 这样的语法，长度只能用立即数表示，所以这里查看 r1 是 5，即 `mr0 5`

```
mr0 5

>-----------------------------------------------------------------------------<
[23:41:37 891]r0=RX@0x400022e0[libhookinunidbg.so]0x22e0, md5=f5704182e75d12316f5b729e89a499df, hex=6c696c6163
size: 5
0000: 6C 69 6C 61 63                                     lilac
^-----------------------------------------------------------------------------^


```

参数 3 是 buffer，我们希望得到函数运算结束后 buffer 字符串的值，但 addBreakPoint 断点触发的时机即函数执行前的时机 (Frida 中叫 OnEnter)，上面我们也说过，断点没有函数的概念，所以我们需要找到等同于 Frida OnLeave 的时机点再下一个断点。

在 ARM 汇编中，LR 寄存器存放了程序的返回地址，当函数跑到 LR 所指向的地址时，不就是函数执行完了吗？Console Debugger 交互调试中使用 blr 命令可以在返回地址处下断点，这就等同于 Frida 的 OnLeave 处的时机。

所以先 _blr_ 下断点，再 _c_ 使得程序继续运行，最后在 lr 地址处再次触发断点，这就是 OnLeave 时机点了。

[![](https://reao.io/usr/uploads/2021/10/21989958.png)](https://reao.io/usr/uploads/2021/10/21989958.png)

[3.png](https://reao.io/usr/uploads/2021/10/21989958.png)

但这里绝不能直接 mr2，因为 R2 只在程序进入口处表示参数 3，在函数运算中，R2 作为通用寄存器被使用和存储其他数据了，而不是 buffer 指针。在 Frida 中也不例外，所以在 OnEnter 中将 args[2] 即 R2 的值，保存在 this.buffer 中。而在 Console Debugger 交互调试中，我们需要更简单粗暴的方法——往上翻一下，看看 OnEnter 处原来 r2 的值是什么。

[![](https://reao.io/usr/uploads/2021/10/2039018631.png)](https://reao.io/usr/uploads/2021/10/2039018631.png)

[4.png](https://reao.io/usr/uploads/2021/10/2039018631.png)

Frida 脚本的等价功能已经完成，是不是感觉有些麻烦？本质上是下两次断点。千万不要觉得麻烦，相信我，在熟练掌握后你会发现 Console Debugger 是最棒最快速最方便的 Hook 调试工具，非常适合辅助算法分析，它也是我目前最常用的 Hook 方式。Console Debugger 还支持很多其他命令，使用起来都很简单，读者可以自行摸索，我们这里再讨论一下参数多于四个的情况该如何查看。首先不论如何，读者都不能把 r4 当成参数 5，把 r5 当成 参数 6，这是完全错误的，ATPCS 规范十分清晰的告诉我们应当查看堆栈。

msp 查看堆栈

[![](https://reao.io/usr/uploads/2021/10/568752122.png)](https://reao.io/usr/uploads/2021/10/568752122.png)

[5.png](https://reao.io/usr/uploads/2021/10/568752122.png)

前四个字节的**小端序**即参数 5，往后偏移四个字节即参数 6，以此类推。即参数 5 = 0xAAAAAAAD ，参数 6 = 0x401D2000。

断点的设置既可以用绝对地址也可以填入使用重载方法填入相对地址，需要注意的是，凡是基于 Unicorn 的 Hook，比如我们现在用的 Console Debugger，都不用管 thumb2 和 arm 模式的区分，不用因为是 Thumb2 就地址 + 1，Unicorn 会替我们处理这个问题。下面这四种方式都下了正确的断点。

```
emulator.attach().addBreakPoint(module.findSymbolByName("base64_encode").getAddress());
emulator.attach().addBreakPoint(module, 0x8A0);
emulator.attach().addBreakPoint(module.base + 0x8A0);
emulator.attach().addBreakPoint(module, 0x8A1);


```

考虑一个问题，如果目标函数调用了三五百次，该怎么获得每一次的参数情况？诚然，下断点，然后不停的触发断点，不停的 `mxx` ，再`c` ，不断循环往复是可以的，但显然麻烦，即我们上述的办法只适用于函数调用少数几次，进行敏捷分析，调用太多次就不太方便。因此需要做 Hook 的持久化。

[![](https://reao.io/usr/uploads/2021/10/4072112943.png)](https://reao.io/usr/uploads/2021/10/4072112943.png)

[6.png](https://reao.io/usr/uploads/2021/10/4072112943.png)

```
emulator.attach().addBreakPoint(module.findSymbolByName("base64_encode").getAddress(), new BreakPointCallback() {
    @Override
    public boolean onHit(Emulator<?> emulator, long address) {
        RegisterContext registerContext = emulator.getContext();
        UnidbgPointer arg0 = registerContext.getPointerArg(0);
        int length = registerContext.getIntArg(1);
        Inspector.inspect(arg0.getByteArray(0, length), "base64_encode input");
        return true;
    }
});


```

看起来很复杂，其实写法十分固定，多加练习即可，onHit 返回 ture 时，断点触发时不会进入 Console Debugger 交互界面，这很符合我们的需求。当函数被调用了三五百次时，我们可以在日志中检索感兴趣的内容。除此之外，稍加修改，代码就变成了更灵活的条件断点，在符合我们需求时断下来进行 Console Debugger 交互调试，方便进行更仔细的分析。

```
emulator.attach().addBreakPoint(module.findSymbolByName("base64_encode").getAddress(), new BreakPointCallback() {
    @Override
    public boolean onHit(Emulator<?> emulator, long address) {
        RegisterContext registerContext = emulator.getContext();
        UnidbgPointer arg0 = registerContext.getPointerArg(0);
        int length = registerContext.getIntArg(1);
        if(length==5){
            Inspector.inspect(arg0.getByteArray(0, length), "base64_encode input");
            return false;
        }else {
            return true;
        }
    }
});


```

不知道你是否发现，上面的脚本仅仅相当于

```
function main(){
    // get base address of target so;
    var base_addr = Module.findBaseAddress("libhookinunidbg.so");

    if (base_addr){
        var func_addr = Module.findExportByName("libhookinunidbg.so", "base64_encode");
        console.log("hook base64_encode function, input:")
        Interceptor.attach(func_addr,{
            // 打印入参
            onEnter: function (args) {
                console.log("\ninput:")
                this.buffer = args[2];
                var length = args[1];
                console.log(hexdump(args[0],{length: length.toUInt32()}))
                console.log("\n")
            },
        })
    }


}

setImmediate(main);


```

我们漏了打印返回值这一步，在 Console Debugger 交互调试中，我们通过 `blr` 在 LR 寄存器下断点，获得 onLeave 的时机，下面在代码中复现这个思路。

```
emulator.attach().addBreakPoint(module.findSymbolByName("base64_encode").getAddress(), new BreakPointCallback() {
    @Override
    public boolean onHit(Emulator<?> emulator, long address) {
        RegisterContext registerContext = emulator.getContext();
        UnidbgPointer arg0 = registerContext.getPointerArg(0);
        final UnidbgPointer arg2 = registerContext.getPointerArg(2);
        int length = registerContext.getIntArg(1);
        Inspector.inspect(arg0.getByteArray(0, length), "base64_encode input");
        // OnLeave
        emulator.attach().addBreakPoint(registerContext.getLRPointer().peer, new BreakPointCallback() {
            @Override
            public boolean onHit(Emulator<?> emulator, long address) {
                String result = arg2.getString(0);
                System.out.println("base64 result:"+result);
                return true;
            }
        });
        return true;
    }
});


```

接下来再考虑一种情况

[![](https://reao.io/usr/uploads/2021/10/2015732452.png)](https://reao.io/usr/uploads/2021/10/2015732452.png)

[7.png](https://reao.io/usr/uploads/2021/10/2015732452.png)

函数 108A4 被引用多处，但我们只关注 DD50 + 6F0 这个引用点的调用，怎么能比较舒服的 Hook 它？如果全部 Hook，其他地方的输出是一种干扰。一种办法是用上文说的条件断点——在 108A4 下断，并限制在 LR = baseAddress + DD50 + 6F0 时才输出数据。

[![](https://reao.io/usr/uploads/2021/10/441640311.png)](https://reao.io/usr/uploads/2021/10/441640311.png)

[8.png](https://reao.io/usr/uploads/2021/10/441640311.png)

还有另外一个办法，更快速方便。直接在我们感兴趣的调用处以及调用处的下一行设置两个断点，那么就是专属于这个调用处的 OnEnter 和 OnLeave 了。

```
emulator.attach().addBreakPoint(module, 0xe440);
emulator.attach().addBreakPoint(module, 0xe444);


```

Console Debugger 在 Hook 这一块的使用基本就说清楚了。Console Debugger 基于 Unicorn Hook，在一些场景下，可能直接用 Unicorn 的 Hook 更直截了当。

[![](https://reao.io/usr/uploads/2021/10/1170544794.png)](https://reao.io/usr/uploads/2021/10/1170544794.png)

[9.png](https://reao.io/usr/uploads/2021/10/1170544794.png)

来看一下新需求，base_encode 是我们所分析的函数，现在我们想查看函数中 0x8A0 处 R0 寄存器的值，0x8A2 处 R2 的值，0x8A4 处 R4 的值。使用三次 addBreakPoint 当然也可以，但 Unicorn 原生 Hook 在这里处理起来更舒服。

hook_add_new 第一个参数是 Hook 回调，我们这里选择 CodeHook，它是逐条 Hook，参数 2 是起始地址，参数 3 是结束地址，参数 4 一般填 null。这意味着从起始地址到终止地址这个范围内的每条指令，我们都可以在其执行前处理它。

```
emulator.getBackend().hook_add_new(new CodeHook() {
        final RegisterContext registerContext = emulator.getContext();
        @Override
        public void hook(Backend backend, long address, int size, Object user) {
            if(address == module.base + 0x8A0){
                int r0 = registerContext.getIntByReg(ArmConst.UC_ARM_REG_R0);
                System.out.println("0x8A0 r0:"+Integer.toHexString(r0));
            }
            if(address == module.base + 0x8A2){
                int r2 = registerContext.getIntByReg(ArmConst.UC_ARM_REG_R2);
                System.out.println("0x8A2 r2:"+Integer.toHexString(r2));
            }
            if(address == module.base + 0x8A4){

                int r4 = registerContext.getIntByReg(ArmConst.UC_ARM_REG_R4);
                System.out.println("0x8A4 r4:"+Integer.toHexString(r4));
            }
        }

        @Override
        public void onAttach(Unicorn.UnHook unHook) {

        }

        @Override
        public void detach() {

        }
    }, module.base + 0x8A0, module.base + 0x8C2, null);
}


```

关于 Unicorn Hook 的基础内容，可以看这篇文章 [Unidbg 的底层支持 - Unicorn](https://bbs.pediy.com/thread-269964.htm) 。上述的逻辑，没有办法通过 Frida 复现，因为 Frida 基于 Inline Hook，原理上决定它没法同时 Hook 很近的几个地址。

接下来是打印 Native 调用栈，Frida 中通过如下语句

```
console.log(Thread.backtrace(this.context, Backtracer.ACCURATE).map(DebugSymbol.fromAddress).join('\n') + '\n');}


```

Unidbg 通过 unwind 技术实现了栈回溯，在 Console Debugger 交互调试中，通过 `bt` 指令查看调用栈

[![](https://reao.io/usr/uploads/2021/10/3530378647.png)](https://reao.io/usr/uploads/2021/10/3530378647.png)

[10.png](https://reao.io/usr/uploads/2021/10/3530378647.png)

在代码中，通过简单一句即可

```
emulator.getUnwinder().unwind();


```

除此之外，针对同一个地址设置多个 Hook 时，反应如下

*   使用 addBreakPoint 方式，只有最后一个生效
*   使用 Unicorn 原生 Hook ，链式 Hook，每个都生效，逐个执行

接下来看一下参数替换 (Replace)，需求如下：当入参是 lilac 时，将其替换成 hello world（别忘了参数 2 代表了加密的长度，要对应修改）。

```
emulator.attach().addBreakPoint(module, 0x8A0, new BreakPointCallback() {
    @Override
    public boolean onHit(Emulator<?> emulator, long address) {
        RegisterContext registerContext = emulator.getContext();
        UnidbgPointer input = registerContext.getPointerArg(0);
        final UnidbgPointer output = registerContext.getPointerArg(2);
        int length = registerContext.getIntArg(1);
        String inputString = input.getString(0);
        String FakeName = "Hello World";
        int FakeLength = FakeName.length();
        MemoryBlock memoryBlock = memory.malloc(FakeLength, true);
        memoryBlock.getPointer().write(FakeName.getBytes(StandardCharsets.UTF_8));

        if(inputString.equals("lilac")){
            emulator.getBackend().reg_write(ArmConst.UC_ARM_REG_R0, memoryBlock.getPointer().peer);
            emulator.getBackend().reg_write(ArmConst.UC_ARM_REG_R1, FakeLength);
        }
        // OnLeave
        emulator.attach().addBreakPoint(registerContext.getLRPointer().peer, new BreakPointCallback() {
            @Override
            public boolean onHit(Emulator<?> emulator, long address) {
                String result = output.getString(0);
                System.out.println("base64 result:"+result);
                return true;
            }
        });
        return true;
    }
});


```

看起来有些复杂，但自己手写几遍也就懂了，再看一个需求——函数替换：这个场景也很多，比如签名校验函数，我们直接替换原有的校验函数，直接返回正确的值。需要注意，这和 Hook 修改返回值是不同的，函数替换时，原函数不执行。

比如 DEMo 中的 verifyApkSign 函数

```
#include <jni.h>
#include <string>
#include "base64.h"
#include <android/log.h>

#define LOGE(...) __android_log_print(ANDROID_LOG_ERROR,"Lilac" ,__VA_ARGS__)

extern "C" void testBase64();
extern "C" int verifyApkSign();

extern "C"
JNIEXPORT void JNICALL
Java_com_example_hookinunidbg_MainActivity_call(JNIEnv *env, jobject thiz) {
    int verifyret = verifyApkSign();
    if(verifyret == 1){
        LOGE("APK sign verify failed!");
    } else{
        LOGE("APK sign verify success!");
    }
    testBase64();
}

extern "C" int verifyApkSign(){
    LOGE("verify apk sign");
    return 1;
};

extern "C" void testBase64(){
    auto *in = (unsigned char *) "lilac";
    unsigned int length = 5;
    char *encode_out;;
    encode_out = static_cast<char *>(malloc(BASE64_ENCODE_OUT_SIZE(length)));
    base64_encode(in, length, encode_out);
}


```

Unidbg 模拟执行，可以发现，我们的 DEMO 里写了一个抽象的 verifyApkSign，并且函数返回 1，进而校验失败。

[![](https://reao.io/usr/uploads/2021/10/3524814.png)](https://reao.io/usr/uploads/2021/10/3524814.png)

[11.png](https://reao.io/usr/uploads/2021/10/3524814.png)

看一下 IDA 中 SO 的反编译情况

[![](https://reao.io/usr/uploads/2021/10/2250938730.png)](https://reao.io/usr/uploads/2021/10/2250938730.png)

[12.png](https://reao.io/usr/uploads/2021/10/2250938730.png)

[![](https://reao.io/usr/uploads/2021/10/664501382.png)](https://reao.io/usr/uploads/2021/10/664501382.png)

[13.png](https://reao.io/usr/uploads/2021/10/664501382.png)

我们希望实现类似 Frida 中 Interceptor.replace 的功能——替换函数，在里面实现自己的逻辑。

```
emulator.attach().addBreakPoint(module.findSymbolByName("verifyApkSign").getAddress(), new BreakPointCallback() {
    RegisterContext registerContext = emulator.getContext();
    @Override
    public boolean onHit(Emulator<?> emulator, long address) {
        System.out.println("替换函数 verifyApkSign");
        emulator.getBackend().reg_write(ArmConst.UC_ARM_REG_PC, registerContext.getLRPointer().peer);
        emulator.getBackend().reg_write(ArmConst.UC_ARM_REG_R0, 0);
        return true;
    }
});


```

我们将 PC 指针赋值为 LR，即函数什么都不执行就直接返回，再将 r0 赋值 0，即返回值为 0。有时候，我们只希望替换目标函数在某处的调用，就好像前面只想 Hook 某处一样，同理——可以根据 LR 判断来源，但也有简单的方法。

[![](https://reao.io/usr/uploads/2021/10/2508154153.png)](https://reao.io/usr/uploads/2021/10/2508154153.png)

[14.png](https://reao.io/usr/uploads/2021/10/2508154153.png)

那就是直接跳过 8CA 这条指令，这样不就实现了 “只影响这一处” 了吗。

```
emulator.attach().addBreakPoint(module, 0x8CA, new BreakPointCallback() {
    RegisterContext registerContext = emulator.getContext();
    @Override
    public boolean onHit(Emulator<?> emulator, long address) {
        System.out.println("nop这里的调用");
        emulator.getBackend().reg_write(ArmConst.UC_ARM_REG_PC, registerContext.getPCPointer().peer + 4 + 1);
        emulator.getBackend().reg_write(ArmConst.UC_ARM_REG_R0, 0);
        return true;
    }
});


```

PC + 4 + 1 可能会让你困惑，+ 4 是为了越过当前跳转指令的四字节，+1 是因为 THUMB2 模式，前面我们说过，Unicorn 原生的 Hook 方式不用管 ARM/THUMB2，即不用考虑 +1，为什么这里又要加 1？这是因为我们在给 PC 硬编码赋值，而不是 Unicorn 帮我们处理，所以需要自己考虑这一点。

当然，在这种场景下，如果返回值只是诸如 01 这种小的数字，更常用的是在 IDA 中使用 KeyPatch 打 Patch

[![](https://reao.io/usr/uploads/2021/10/1823857060.png)](https://reao.io/usr/uploads/2021/10/1823857060.png)

[15.png](https://reao.io/usr/uploads/2021/10/1823857060.png)

在 SO 文件中打 Patch 没什么难的，我们也可以在 Unidbg 虚拟内存里打 Patch。首先查一下 mov r0,0 的机器码。

[![](https://reao.io/usr/uploads/2021/10/2972401779.png)](https://reao.io/usr/uploads/2021/10/2972401779.png)

[16.png](https://reao.io/usr/uploads/2021/10/2972401779.png)

```
int patchCode = 0x4FF00000; // movs r0,0
emulator.getMemory().pointer(module.base + 0x8CA).setInt(0,patchCode);


```

上述两行代码就行了。

但每次都查看 ARMConverter 多少有点不够优雅，也不够灵活，Unidbg 里可以用 KeyStone。

```
try (Keystone keystone = new Keystone(KeystoneArchitecture.Arm, KeystoneMode.ArmThumb)) {
    KeystoneEncoded encoded = keystone.assemble("mov r0,0");
    byte[] patchCode = encoded.getMachineCode();
    emulator.getMemory().pointer(module.base + 0x8CA).write(0, patchCode, 0, patchCode.length);
}


```

至此，我们已经基本讲清了 Unidbg 中基于 Unicorn 的 Hook，为什么还要学其他方式的 Hook？

[![](https://reao.io/usr/uploads/2021/10/423743050.png)](https://reao.io/usr/uploads/2021/10/423743050.png)

[17.png](https://reao.io/usr/uploads/2021/10/423743050.png)

很大一个原因就是 Unidbg 以及基于 Unidbg 的 Hook 只能在 Unidbg 使用 Unicorn 引擎时才能用（除了上面的 patch），但在模拟执行时，一般都会切换成 Dynarmic 引擎，因为这个引擎比 Unicorn 快很多倍，生产环境中，肯定速度越快越好，因为我们得考虑不能使用 Unidbg Hook 情况下的 Hook 工具。即 Unidbg 辅助算法分析时用 Unicorn 引擎 + Console Debugger，而模拟执行时用 Dynarmic + 其他 Hook 工具。

有些人会困惑，为什么模拟执行时要用到 Hook 呢？打个比方，我需要随机化某个函数的输出，对抗风控，或者我需要修改某个执行逻辑，完成自定义的功能，这些场景是真实存在的。

首先介绍 xHook，它是爱奇艺开源的 PLT hook 库，受制于原理，无法 Hook 内部函数（IDA 里的 Sub_xxx），但在它能 Hook 的函数上，稳定性比较好。

Hook base64_encode 打印入参

```
IxHook ixHook = XHookImpl.getInstance(emulator);
ixHook.register(module.name, "base64_encode", new ReplaceCallback() {
    @Override
    public HookStatus onCall(Emulator<?> emulator, long originFunction) {
        String str = emulator.getContext().getPointerArg(0).getString(0);
        System.out.println("base64 input:"+str);
        return HookStatus.RET(emulator, originFunction);
    }
});
// 使其生效
ixHook.refresh();


```

接下来介绍 HookZz，HookZz 现在叫 Dobby，Unidbg 中是两个独立的 Hook 库，因为作者认为 HookZz 在 arm32 上支持较好，Dobby 在 arm64 上稳定性较好。HookZz 是 inline hook 方案，因此可以 Hook Sub_xxx，也可以 Hook 某一行汇编。

Hook base64_encode 打印入参和 buffer 的返回值

```
IHookZz hookZz = HookZz.getInstance(emulator); // 加载HookZz，支持inline hook
    hookZz.enable_arm_arm64_b_branch(); // 测试enable_arm_arm64_b_branch，可有可无
    hookZz.wrap(module.findSymbolByName("base64_encode"), new WrapCallback<HookZzArm32RegisterContext>() {
        @Override
        public void preCall(Emulator<?> emulator, HookZzArm32RegisterContext ctx, HookEntryInfo info) {
            Pointer pointer = ctx.getPointerArg(0);
            int length = ctx.getIntArg(1);
            byte[] input = pointer.getByteArray(0, length);
            Inspector.inspect(input, "base64 input");
            // 把参数二存在ctx里，等到postCall即Frida的OnLeave中再取出来
            ctx.push(ctx.getPointerArg(2));
        }
        @Override
        public void postCall(Emulator<?> emulator, HookZzArm32RegisterContext ctx, HookEntryInfo info) {
            Pointer result = ctx.pop();
            System.out.println("base64 result:" + result.getString(0));
        }
    });
    hookZz.disable_arm_arm64_b_branch();


```

修改 verifyApkSign 函数返回值为 0

```
hookZz.wrap(module.findSymbolByName("verifyApkSign"), new WrapCallback<EditableArm32RegisterContext>() {
    @Override
    public void preCall(Emulator<?> emulator, EditableArm32RegisterContext ctx, HookEntryInfo info) {
        System.out.println("onEnter verify");
    }
    @Override
    public void postCall(Emulator<?> emulator, EditableArm32RegisterContext ctx, HookEntryInfo info) {
        System.out.println("OnLeave verify");
        // 修改返回值为0
        ctx.setR0(0);
    }
});


```

替换 verifyApkSign 函数并返回 0

```
hookZz.replace(module.findSymbolByName("verifyApkSign"), new ReplaceCallback() {
    @Override
    public HookStatus onCall(Emulator<?> emulator, HookContext context, long originFunction) {
        emulator.getBackend().reg_write(Unicorn.UC_ARM_REG_R0,0);
        return HookStatus.LR(emulator, context.getLR());
    }
});


```

除此之外，也可以实现类似于 Unicorn 断点性质的 Hook

```
hookZz.instrument(module.base + 0x008ce + 1, new InstrumentCallback<RegisterContext>() {
    @Override
    public void dbiCall(Emulator<?> emulator, RegisterContext ctx, HookEntryInfo info) {
        System.out.println(ctx.getIntArg(0));
    }
});


```

可以发现，在一些场景下，HookZz 等第三方 Hook 工具的写法更简单，但我个人的观点是在非必要的情况下，不使用第三方 Hook 工具。

原因有三：

*   HookZz 或者 xHook 等等 Hook 方案，都可以基于其原理进行检测，但 Unicorn 原生 Hook 不容易被检测。

[SliverBullet5563/CheckGotHook: 检测 got hook（使用 xhook 测试）](https://github.com/SliverBullet5563/CheckGotHook)

[liaogang/check_fish_inline_hook: fishhook 和 inlinehook 的检测](https://github.com/liaogang/check_fish_inline_hook)

*   原生 Hook 在 Hook 时不会出问题，其他 Hook 方案都可能出 BUG 或者本身就不具备那种能力，比如 Inline Hook 方案不能 Hook 短函数，或者 Hook 两个相邻的地址。
*   在辅助算法分析时，第三方 Inline Hook + 原生 Hook 同时使用时会出 BUG，事实上，只单使用 Unicorn 的 Memory Hook 功能都会 BUG，因此统一用原生 Hook 会少部分麻烦。

### 三、杂记

#### 0、时机问题

如果目标函数在 init_proc 或者 init_array 中，JNIOnLoad 前下断或者 Hook 的这个时机点就显得晚了，init 的执行时机在 vm.loadLibrary 函数中。但好在 Unidbg 不存在 ALSR（地址随机化），因此可以先确认一下目标 SO 的加载基地址，然后在 loadLibrary 之前下断点，即可保证最早的时机，任何调用都不会错过。

```
emulator.attach().addBreakPoint(0x40000000+0xa80);

// 加载so到虚拟内存
DalvikModule dm = vm.loadLibrary("hookinunidbg", true);


```

#### 1、读写 std::string

Frida 里这么做

```
function readStdString(str){
  var isTiny = (str.readU8 & 1) === 0;
  if (isTiny){
    return str.add(1).readUtf8String();
  }
  return str.add(2 * Process.pointerSize).readPointer().readUtf8String();
}



function writeStdString(str, content){
  var isTiny = (str.readU8() & 1) === 0;
  if (isTiny){
    str.add(1).writeUtf8String(content);
  }else{
    str.add(2 * Process.pointerSize).readPointer().writeUtf8String(content);
  }
}


```

Unidbg 中如法炮制

```
public String readStdString(Pointer strptr){
    Boolean isTiny = (strptr.getByte(0) & 1) == 0;
    if(isTiny){
        return strptr.getString(1);
    }
    return strptr.getPointer(emulator.getPointerSize()* 2L).getString(0);
}

public void writeStdString(Pointer strptr, String content){
    Boolean isTiny = (strptr.getByte(0) & 1) == 0;
    if(isTiny){
        strptr.write(1, content.getBytes(StandardCharsets.UTF_8), 0, content.length());
    }
    strptr.getPointer(emulator.getPointerSize()* 2L).write(0, content.getBytes(StandardCharsets.UTF_8), 0, content.length());
};


```

#### 2、读 jobject，诸如 jstring

// TODO

#### 3、TraceAllFunctions

在 Frida 上，可以使用 [trace_natives](https://github.com/Pr0214/trace_natives) 对 一个 SO 中的全部函数进行 Trace，并形成如下调用图，在一些场景中，可以帮助加快分析速度。

[![](https://reao.io/usr/uploads/2021/10/287503166.png)](https://reao.io/usr/uploads/2021/10/287503166.png)

[18.png](https://reao.io/usr/uploads/2021/10/287503166.png)

脚本很简单，我们只需要理解三个点

*   如何获得一个 SO 的全部函数其偏移地址
*   如何 Hook 函数
*   如何获得调用层级关系

问题 1 的解决办法：IDA 会解析 SO 中的全部函数，非导出函数会被解析成 sub_xxx，可以使用 IDAPython 编写脚本获取函数列表。

```
def getFunctionList():
    functionlist = ""
    minLength = 10
    maxAddress = ida_ida.inf_get_max_ea()
    for func in idautils.Functions(0, maxAddress):
        if len(list(idautils.FuncItems(func))) > minLength:
            functionName = str(idaapi.ida_funcs.get_func_name(func))
            oneFunction = hex(func) + "!" + functionName + "\t\n"
            functionlist += oneFunction
    return functionlist


```

脚本获取了函数以及对应函数名列表，同时通过 minLength 过滤较短的函数，至少包含 10 条汇编指令的函数才会被计入。这么做有两个原因

*   过短的函数可能导致 Frida Hook 失败（inline hook 原理所致）
*   过短的函数可能是工具函数，调用次数多，但价值不大，让调用图变得臃肿不堪

完整的 IDA 脚本如下，作用就是在 SO 路径下生成一个记录了 SO 中全部函数的文本文件。

```
import os
import time

import ida_ida
import ida_nalt
import idaapi
import idautils
from idaapi import plugin_t
from idaapi import PLUGIN_PROC
from idaapi import PLUGIN_OK


def getFunctionList():
    functionlist = ""
    minLength = 10
    maxAddress = ida_ida.inf_get_max_ea()
    for func in idautils.Functions(0, maxAddress):
        if len(list(idautils.FuncItems(func))) > minLength:
            functionName = str(idaapi.ida_funcs.get_func_name(func))
            oneFunction = hex(func) + "!" + functionName + "\t\n"
            functionlist += oneFunction
    return functionlist


# 获取SO文件名和路径
def getSoPathAndName():
    fullpath = ida_nalt.get_input_file_path()
    filepath, filename = os.path.split(fullpath)
    return filepath, filename


class getFunctions(plugin_t):
    flags = PLUGIN_PROC
    comment = "getFunctions"
    help = ""
    wanted_name = "getFunctions"
    wanted_hotkey = ""

    def init(self):
        print("getFunctions(v0.1) plugin has been loaded.")
        return PLUGIN_OK

    def run(self, arg):

        so_path, so_name = getSoPathAndName()
        functionlist = getFunctionList()
        script_name = so_name.split(".")[0] + "_functionlist_" + str(int(time.time())) + ".txt"
        save_path = os.path.join(so_path, script_name)
        with open(save_path, "w", encoding="utf-8") as F:
            F.write(functionlist)
        F.close()
        print(f"location: {save_path}")

    def term(self):
        pass


def PLUGIN_ENTRY():
    return getFunctions()


```

问题 2 的解决办法：使用 Frida 啦当然是。

问题 3 的解决办法：Frida 的 Frida-trace 可以自动展现这种层级关系，所以 traceNatives 脚本依赖 Frida-trace 完成了展示图。可是 Frida-trace 是怎么做到的？分析源码发现，使用了 Frida 的 Interceptor.attach 环境中的 depth（如下代码中的 this.depth）。depth 表示调用深度，其依赖于 Frida 的栈回溯。

```
Interceptor.attach(Module.getExportByName(null, 'read'), {
  onEnter(args) {
    console.log('Context information:');
    console.log('Context  : ' + JSON.stringify(this.context));
    console.log('Return   : ' + this.returnAddress);
    console.log('ThreadId : ' + this.threadId);
    console.log('Depth    : ' + this.depth);
    console.log('Errornr  : ' + this.err);

    // Save arguments for processing in onLeave.
    this.fd = args[0].toInt32();
    this.buf = args[1];
    this.count = args[2].toInt32();
  },
  onLeave(result) {
    console.log('----------')
    // Show argument 1 (buf), saved during onEnter.
    const numBytes = result.toInt32();
    if (numBytes > 0) {
      console.log(hexdump(this.buf, { length: numBytes, ansi: true }));
    }
    console.log('Result   : ' + numBytes);
  }
})


```

这三个问题能在 Unidbg 中解决吗？来试试 Unidbg 版本的 TraceNatives？

问题 1：IDA 插件而已，和使用 Frida 或者 Unidbg 无关。

问题 2：使用 Unidbg 原生 Hook，效果比原版用 Frida 会更好，因为短函数或很密集的函数也能 Hook。

问题 3：Unidbg 也实现了 arm unwind 栈回溯

我们不妨再思考一下，问题一的答案。TraceNatives 项目中借助 IDA 实现对 SO 函数的识别。但借助 IDA 也增加了复杂度和使用成本，这儿不妨不用 IDA，换个办法识别函数。ARM 中，函数常常以 push 指令开始，配合 Unidbg 的 BlockHook，就可以识别绝大多数函数，少部分会遗漏，但也无可奈何嘛。BlockHook 还会提供当前块的大小，对于较小的块不予理睬。

接下来处理问题三，Unidbg 提供了栈回溯，但没有提供打印调用深度的函数，在 Unwinder 类中添加一个函数

_src/main/java/com/github/unidbg/unwind/Unwinder.java_

```
public final int depth(){
    int count = 0;
    Frame frame = null;
    while((frame = unw_step(emulator, frame)) != null) {
        if(frame.isFinish()){
            return count;
        }
        count++;
    }
    return count;
}


```

接下来大功告成，组装一下代码

```
PrintStream traceStream = null;
try {
    // 保存文件
    String traceFile = "unidbg-android/src/test/resources/app/traceFunctions.txt";
    traceStream = new PrintStream(new FileOutputStream(traceFile), true);
} catch (FileNotFoundException e) {
    e.printStackTrace();
}

final PrintStream finalTraceStream = traceStream;
emulator.getBackend().hook_add_new(new BlockHook() {
    @Override
    public void hookBlock(Backend backend, long address, int size, Object user) {
        if(size>8){
            Capstone.CsInsn[] insns = emulator.disassemble(address, 4, 0);
            if(insns[0].mnemonic.equals("push")){
                int level = emulator.getUnwinder().depth();
                assert finalTraceStream != null;
                for(int i = 0 ; i < level ; i++){
                    finalTraceStream.print("    |    ");
                }
                Symbol symbol = module.findClosestSymbolByAddress(address, true);
                if(symbol !=null){
                    System.out.println(symbol.getName());
                }

                finalTraceStream.println("  "+"sub_"+Integer.toHexString((int) (address-module.base))+"  ");
            }
        }

    }

    @Override
    public void onAttach(Unicorn.UnHook unHook) {

    }

    @Override
    public void detach() {

    }
}, module.base, module.base+module.size, 0);


```

可以发现代码非常的简洁优雅。

#### **4、Unidbg Call Everything**

// TODO

#### 5、Print A in B

首先写一个 DEMO, native.cpp 代码如下

```
#include <jni.h>
#include <string>

extern "C" void function1(){
    char str1[30] = "In function1";
    char str2[30] = "";
    strcmp(str1, str2);
}


extern "C" void function2(){
    char str1[30] = "In function2";
    char str2[30] = "";
    strcmp(str1, str2);
}

extern "C" JNIEXPORT jstring JNICALL
Java_com_example_reader_MainActivity_stringFromJNI(
        JNIEnv* env,
        jobject /* this */) {
    function1();
    function2();
    std::string hello = "Hello from C++";
    return env->NewStringUTF(hello.c_str());
}


```

MainActivity.java 代码如下

```
package com.example.reader;

import androidx.appcompat.app.AppCompatActivity;

import android.os.Bundle;
import android.view.View;
import android.widget.Button;
import android.widget.TextView;

import com.example.reader.databinding.ActivityMainBinding;

public class MainActivity extends AppCompatActivity {

    // Used to load the 'reader' library on application startup.
    static {
        System.loadLibrary("reader");
    }

    private ActivityMainBinding binding;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        binding = ActivityMainBinding.inflate(getLayoutInflater());
        setContentView(binding.getRoot());

        // Example of a call to a native method
        TextView tv = binding.sampleText;
        tv.setText("");

        Button btn = findViewById(R.id.button);
        btn.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View view) {
                stringFromJNI();
            }
        });
    }

    /**
     * A native method that is implemented by the 'reader' native library,
     * which is packaged with this application.
     */
    public native String stringFromJNI();


}


```

接下来使用 Frida Hook Strcmp 函数

```
Interceptor.attach(
    Module.findExportByName("libc.so", "strcmp"), {
        onEnter: function(args) {
            console.log("strcmp arg1:"+args[0].readCString())
        },
        onLeave: function(ret) {
        }
    }
);


```

每次点击按钮，都会触发上百次 strcmp 调用，**进程中所有对 strcmp 的调用都被捕获了**。

对应的 Unidbg 代码如下

```
public void hookstrcmp(){
    long address = module.findSymbolByName("strcmp").getAddress();
    System.out.println("strcmp address:"+Integer.toHexString((int) address));
    emulator.attach().addBreakPoint(address, new BreakPointCallback() {
        @Override
        public boolean onHit(Emulator<?> emulator, long address) {
            RegisterContext registerContext = emulator.getContext();
            String arg1 = registerContext.getPointerArg(0).getString(0);
            System.out.println("strcmp arg1:"+arg1);
            return true;
        }
    });
}


```

但这存在一个问题——干扰信息太多（frida）。所以我们希望做一下过滤，判断 strcmp 由哪个 SO 调用，进而可以只打印某个 SO 中发生的 strcmp。即实现 “**只打印发生在某个 SO 中的 strcmp 调用**”。

```
Interceptor.attach(
    Module.findExportByName("libc.so", "strcmp"), {
        onEnter: function(args) {
            var moduleName = Process.getModuleByAddress(this.returnAddress).name;
            console.log("strcmp arg1:"+args[0].readCString())
            console.log("call from :"+moduleName)
        },
        onLeave: function(ret) {
        }
    }
);


```

对应的 Unidbg 代码如下

```
public void hookstrcmp(){
    long address = module.findSymbolByName("strcmp").getAddress();
    emulator.attach().addBreakPoint(address, new BreakPointCallback() {
        @Override
        public boolean onHit(Emulator<?> emulator, long address) {
            RegisterContext registerContext = emulator.getContext();
            String arg1 = registerContext.getPointerArg(0).getString(0);
            String moduleName = emulator.getMemory().findModuleByAddress(registerContext.getLRPointer().peer).name;
            if(moduleName.equals("libreader.so")){
                System.out.println("strcmp arg1:"+arg1);
            }
            return true;
        }
    });
}


```

再进一步，如果我们在分析某个关键函数 A，希望只打印 A 中对 B 函数的调用，该怎么做呢？比如只想 Hook function1 中发生 strcmp。

```
var show = false;
Interceptor.attach(
    Module.findExportByName("libc.so", "strcmp"), {
        onEnter: function(args) {
            if(show){
                console.log("strcmp arg1:"+args[0].readCString())
            }
        },
        onLeave: function(ret) {

        }
    }
);

Interceptor.attach(
    Module.findExportByName("libreader.so", "function1"),{
        onEnter: function(args) {
            show = this;
        },
        onLeave: function(ret) {
            show = false;
        }
    }
)


```

在 Unidbg 中显得复杂一些

```
// 声明全局变量 public boolean show = false;
public void hookstrcmp(){
    emulator.attach().addBreakPoint(module.findSymbolByName("function1").getAddress(), new BreakPointCallback() {
        @Override
        public boolean onHit(Emulator<?> emulator, long address) {
            RegisterContext registerContext = emulator.getContext();

            show = true;
            emulator.attach().addBreakPoint(registerContext.getLRPointer().peer, new BreakPointCallback() {
                @Override
                public boolean onHit(Emulator<?> emulator, long address) {
                    show = false;
                    return true;
                }
            });
            return true;
        }
    });

    emulator.attach().addBreakPoint(module.findSymbolByName("strcmp").getAddress(), new BreakPointCallback() {
        @Override
        public boolean onHit(Emulator<?> emulator, long address) {
            RegisterContext registerContext = emulator.getContext();
            String arg1 = registerContext.getPointerArg(0).getString(0);
            if(show){
                System.out.println("strcmp arg1:"+arg1);
            }
            return true;
        }
    });
}


```

如上所示，在一些情况下可以减少干扰。