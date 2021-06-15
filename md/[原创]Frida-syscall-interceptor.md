> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-268086.htm)

> [原创]Frida-syscall-interceptor

[](#一、目标)一、目标
-------------

现在很多 App 不讲武德了，为了防止 **openat 、read、kill** 等等底层函数被 hook，干脆就直接通过 syscall 的方式来做系统调用，导致无法 hook。

 

应对这种情况有两种方案：

*   刷机重写系统调用表来拦截内核调用
*   inline Hook SWI/SVC 指令

我们今天采用第二种方法，用 frida 来实现

*   内联汇编 SWI/SVC 做系统调用， syscall
*   frida inline hook
*   hook syscall
*   frida ArmWriter
*   frida typescript project

[](#二、步骤)二、步骤
-------------

### inline Hook 原理

. 备份 SWI/SVC 部分的指令，重写成为跳转指令

 

. 跳转到我们新的代码空间，把之前备份的指令执行一下。然后执行我们自己的逻辑。 (打印参数之类的)

 

. 跳回原程序空间，继续往下跑

 

![](https://bbs.pediy.com/upload/attach/202106/907124_3QDVXWPXJ5P44GQ.jpg)

### 重写成为跳转指令

这次 hook 使用 frida ArmWriter 来实现，用 **putBranchAddress** 函数写个跳转指令，需要花 20 个字节，我们先看看修改之前的情况。

 

![](https://bbs.pediy.com/upload/attach/202106/907124_H7TUZYPM4N4BCMY.png)

```
// 我们定位的锚是 svc指令， 0x9374C1F8  它前面还有8个字节的指令这里一并替换
const address = syscallAddress.sub(8);
// 备份这20个字节，马上它们就要被替换了
const instructions = address.readByteArray(20);
 
if (instructions == null) {
    throw new Error(`Unable to read instructions at address ${address}.`);
}
 
// 把旧的20个字节打印出来
console.log(" ==== old instructions ==== " + address);
console.log(instructions);
 
// 开始替换成跳转指令，跳转的地址是 createCallback 里面创建的新的代码空间地址。
Memory.patchCode(address, 20, function (code) {
 
    let writer = null;
 
    writer = new ArmWriter(code, { pc: address });
    writer.putBranchAddress(createCallback(callback, instructions, address.add(20), syscallAddress));
    writer.flush();
});
 
// 把新的指令打出来对比下
console.log(" ==== new instructions ==== " + address);
const instructionsNew = address.readByteArray(20);
console.log(instructionsNew);

```

跑一下结果

```
==== old instructions ==== 0x937621f0
           0  1  2  3  4  5  6  7  8  9  A  B  C  D  E  F  0123456789ABCDEF
00000000  07 c0 a0 e1 42 71 00 e3 00 00 00 ef 0c 70 a0 e1  ....Bq.......p..
00000010  01 0a 70 e3                                      ..p.
 ==== new Code Addr ====
0xa9b83000
 ==== retAddress ====
0x93762204
 ==== new instructions ==== 0x937621f0
           0  1  2  3  4  5  6  7  8  9  A  B  C  D  E  F  0123456789ABCDEF
00000000  01 80 2d e9 04 00 9f e5 04 00 8d e5 01 80 bd e8  ..-.............
00000010  00 30 b8 a9                                      .0..

```

指令是修改成功了，但是修改的对不对呢？ 这时候需要祭出 IDA，Attach 一下我们的 demo。对比一下。

 

![](https://bbs.pediy.com/upload/attach/202106/907124_V53FCXN3DN58JX6.png)

 

新的代码空间地址是 0xa9b83000。 从我们修改的指令来看，它会跳转到 0xa9b83000， 木有问题。

 

然后返回的地址是 0x93762204， 恰好也是要回来的地址。

### [](#执行备份指令，和我们自己的逻辑)执行备份指令，和我们自己的逻辑

执行备份指令比较简单，但是我们自己的逻辑可不能用 Arm 汇编来写，frida 已经帮我们想好了，可以创建一个 **NativeCallback**， 执行备份指令之后，直接可以跳转到 firida 的 **NativeCallback**。 听起来很牛的样子。

```
// Hook 逻辑，这里只打印 参数
hookSyscall(address, new NativeCallback(function (dirfd, pathname, mode, flags) {
    let path = pathname.readCString();
 
    log('Called openat hook');
    log('- R0: ' + dirfd);
    log('- R1: ' + path);
    log('- R2: ' + mode);
    log('- R3: ' + flags);
 
    return 0;
}, 'int', ['int', 'pointer', 'int', 'int']));
 
 
......
 
 
// 创建一个新的代码空间，放我们自己的代码
let frida = Memory.alloc(Process.pageSize);
 
// 开始写程序了
writer = new ArmWriter(code, { pc: frida });
 
// 执行备份的指令
writer.putBytes(instructions);
 
// 寄存器入栈，这里把r0也入栈了
// FF 5F 2D E9 STMFD  SP!, {R0-R12,LR} 寄存器入栈 
writer.putInstruction(0xE92D5FFF);
// 00 A0 0F E1 MRS R10, CPSR
// 00 04 2D E9 STMFD SP!, {R10}    // 状态寄存器入栈
writer.putInstruction(0xE10FA000);
writer.putInstruction(0xE92D0400);
 
// instructions.size = 20  + 5条指令
// 修改lr寄存器，保障执行我们自己的逻辑之后还能回来继续向下执行。
writer.putLdrRegAddress("lr",frida.add(20 + 5*4));
writer.putBImm(callback);
 
// 00 04 BD E8  LDMFD SP!, {R10}   // 状态寄存器出栈    
// 0A F0 29 E1  MSR CPSR_cf, R10
writer.putInstruction(0xE8BD0400);
writer.putInstruction(0xE129F00A);
 
// FF 5F BD E8 LDMFD  SP!, {R0-R12,LR}    寄存器出栈
writer.putInstruction(0xE8BD5FFF);
 
// 我回来了 0x93762204
writer.putBranchAddress(retAddress);
writer.flush();

```

再跑一下，

```
Called openat hook
- R0: 86
- R1: /proc/self/maps                                                                                 
- R2: 0
- R3: 0

```

[](#三、总结)三、总结
-------------

本文来自 [https://github.com/AeonLucid/frida-syscall-interceptor](https://github.com/AeonLucid/frida-syscall-interceptor) ，(对，就是 AndroidNativeEmu 的作者。 Orz) , **作者实现了 arm64 下面的 hook，我们把 arm32 的 hook 补上了**，所以调试的时候需要在 arm32 的手机上去调试。

 

frida 脚本采用 typescript project, 调试和编译脚本的时候需要参照 [https://github.com/oleavr/frida-agent-example](https://github.com/oleavr/frida-agent-example) 。

 

frida-syscall-interceptor 和 frida-agent-example 在同级目录。

 

生成 js 文件时，会提示 ArmWriter 的 putBranchAddress 函数找不到，其实这个函数是存在的，只是库文件没有更新， 手工在 **declare class ArmWriter** 里面增加一下 putBranchAddress 的声明。

 

我们的实现只是把参数和结果打印出来了，在我们自己的 **NativeCallback** 中并不能方便的去修改这些入参和返回值。这个就给大家留作业了。 参照 [https://github.com/zhuotong/Android_InlineHook](https://github.com/zhuotong/Android_InlineHook) 的原理，那就想怎么玩就怎么玩。

[第五届安全开发者峰会（SDC 2021）议题征集正式开启！](https://bbs.pediy.com/thread-266645.htm)

上传的附件：

*   [fridaSrc.zip](javascript:void(0)) （297.20kb，6 次下载）
*   [syscalldemoAPK.zip](javascript:void(0)) （2.66MB，6 次下载）