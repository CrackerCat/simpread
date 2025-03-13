> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-286006.htm)

> [原创] 某加固 onCreate 的 vmp 分析

某加固 onCreate 的 Vmp 分析
=====================

前言
--

复现完 oacia 师傅的文章，[360 加固 dex 解密流程分析 | oacia = oacia の BbBlog~ = DEVIL or SWEET](elink@681K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6G2j5h3y4A6j5g2)9J5k6h3c8W2N6W2)9J5c8U0x3$3x3q4)9J5k6r3A6A6j5h3N6#2i4K6u0r3)

想继续来看看 onCreate 怎么实现的，继续使用 oacia 师傅的加固 app

定位 onCreate 函数
--------------

onCreate 函数被注册为 native 函数

![](https://cdn.jsdelivr.net/gh/1ens/blogImages/202503011747515.png)

Init 函数，attachBaseContext 函数，都没有什么可用的线索。发现 static 函数里有一个 jni 函数 interface11，

![](https://cdn.jsdelivr.net/gh/1ens/blogImages/202503131627650.png)

在 attachBaseContext 执行之前。所有 `static {}` 代码块会在类加载时执行（`Application` 类的 `constructor` 之前)

使用 yang 神的脚本，拦截 JNI 注册函数 RegisterNative 找到函数的地址

```
[RegisterNatives] java_class: com.stub.StubApp name: interface11 sig: (I)V fnPtr: 0x73fd0bb9d8 module_name: libjiagu_64.so module_base: 0x73fcf82000 offset: 0x1399d8
```

直接来看汇编代码, 前面都是一些线程检测，以及寄存器操作等，来到最后跳转的是 x8 寄存器的值，没法直接反编译

![](https://cdn.jsdelivr.net/gh/1ens/blogImages/202503011846000.png)

用 stalker，跟踪 x8 寄存器的变化，来打印跳转

```
function hookTargetFunc() {
    var baseAddr = Module.findBaseAddress(TargetLibName)
    //console.log(TargetLibName + " Module Base: " + baseAddr);
    if (baseAddr == null) {
        console.log('Please makesure ' + TargetLibName + ' is Loaded, by setting extractNativeLibs true.');
        return;
    }
    // hook指定的函数地址
    Interceptor.attach(baseAddr.add(TargetFuncOffset), {
        onEnter: function (args) {
            console.log('\nCall ' + TargetFuncOffset.toString(16).toUpperCase() + ' In')
            this.tid = Process.getCurrentThreadId();
            var so_addr = Module.findBaseAddress(so_name);
            var so_size = Process.getModuleByName(so_name).size;
            Stalker.follow(this.tid, {
                events: {
                    call: true, // CALL instructions: yes please
                    // Other events:
                    ret: false, // RET instructions
                    exec: false, // all instructions: not recommended as it's
                    //                   a lot of data
                    block: false, // block executed: coarse execution trace
                    compile: false // block compiled: useful for coverage
                },
                onReceive: function () {
                  },
 
                transform: function (iterator) {
                    var instruction = iterator.next();
                    const startAddress = instruction.address;
                    if (startAddress){
                        do{
                               if (instruction.mnemonic.startsWith('br')||instruction.mnemonic.startsWith('blr')||instruction.mnemonic.startsWith('bl')) {
 
                                    try {
 
                                        iterator.putCallout(function (context) {
                                            var pc = context.pc; //获取pc寄存器，当前地址
                                            var lr = context.lr;
                                            var x8 = context.x8; //获取x8寄存器，
                                            var module = Process.findModuleByAddress(pc);
                                            if (module) {
                                                try{
                                                  console.log(module.name + "!" +Memory.readCString(ptr(x8)));  
                                                }
                                                catch (e){
                                                }                                            
                                                console.log("x8 ====" + DebugSymbol.fromAddress(ptr(x8)));
                                            }
                                        });
                                } catch (e) {
                                    console.log("error", e)
                                }
                             }
                            iterator.keep();
                        } while ((instruction = iterator.next()) !== null);
                    }
              }, 
                onCallSummary(summary) {
                }
            })
        }, onLeave: function (retVal) {
            console.log('Call ' + TargetFuncOffset.toString(16).toUpperCase() + ' Out\n')
            Stalker.unfollow(this.tid)
        }
    })
}
```

根据 trcace 发现了两处 oncreate，看看的后一个地址干了什么

```
x8 ====0x7bfee7fe0c libjiagu_64.so!0x139e0c
x8 ====0x7bfee7fe64 libjiagu_64.so!0x139e64
x8 ====0x7bfee7fe7c libjiagu_64.so!0x139e7c//似乎是重点的
x8 ====0x7bfee80460 libjiagu_64.so!0x13a460//拷贝操作
x8 ====0x7bfee7fe8c libjiagu_64.so!0x139e8c
x8 ====0x7bfee7ff60 libjiagu_64.so!0x139f60 //来到复杂函数 sub_19DCA4
x8 ====0x7bfee7ffa0 libjiagu_64.so!0x139fa0//拷贝
x8 ====0x7bfee7ffc4 libjiagu_64.so!0x139fc4
x8 ====0x7bfee8010c libjiagu_64.so!0x13a10c
x8 ====0x7bfee80140 libjiagu_64.so!0x13a140
x8 ====0x7bfee80170 libjiagu_64.so!0x13a170
//oncreate
x8 ====0x7bfee80194 libjiagu_64.so!0x13a194
x8 ====0x7bfee801fc libjiagu_64.so!0x13a1fc
x8 ====0x7bfee804e4 libjiagu_64.so!0x13a4e4
x8 ====0x7bfee8024c libjiagu_64.so!0x13a24c
x8 ====0x7bfee80264 libjiagu_64.so!0x13a264//进程检测mutex_lock
//前面似乎一些栈的操作，然后寄存器发生变化
x8 ====0x7bfee802e8 libjiagu_64.so!0x13a2e8//
x8 ====0x7bfee80328 libjiagu_64.so!0x13a328//内存
x8 ====0x7bfee7fe64 libjiagu_64.so!0x139e64
x8 ====0x7bfee80600 libjiagu_64.so!0x13a600
x8 ====0x7bfee8061c libjiagu_64.so!0x13a61c
//然后出现下一个oncreate
```

![](https://cdn.jsdelivr.net/gh/1ens/blogImages/202503020857106.png)

后一个 oncreate, 都是一些内存和栈的操作。

```
x8 ====0x7bfee807ac libjiagu_64.so!0x13a7ac//下一步跳转
x8 ====0x7bfee807b8 libjiagu_64.so!0x13a7b8
x8 ====0x7bfee807f4 libjiagu_64.so!0x13a7f4
x8 ====0x7bfee80808 libjiagu_64.so!0x13a808
x8 ====0x7bfee80810 libjiagu_64.so!0x13a810
x8 ====0x7bfee8082c libjiagu_64.so!0x13a82c
x8 ====0x7bfee80854 libjiagu_64.so!0x13a854
x8 ====0x7bfee80854 libjiagu_64.so!0x13a854
x8 ====0x7bfee80838 libjiagu_64.so!0x13a838
x8 ====0x7bfee807e0 libjiagu_64.so!0x13a7e0
x8 ====0x7bfee807f4 libjiagu_64.so!0x13a7f4
x8 ====0x7bfee80870 libjiagu_64.so!0x13a870
x8 ====0x7bfee8087c libjiagu_64.so!0x13a87c
x8 ====0x7bfee80894 libjiagu_64.so!0x13a894
x8 ====0x7bfee80894 libjiagu_64.so!0x13a894
x8 ====0x7bfee80894 libjiagu_64.so!0x13a894
x8 ====0x7bfee80894 libjiagu_64.so!0x13a894
x8 ====0x7bfee808c4 libjiagu_64.so!0x13a8c4
x8 ====0x7bfee808dc libjiagu_64.so!0x13a8dc
x8 ====0x7bfee808e8 libjiagu_64.so!0x13a8e8
```

0x13a170 函数之后出现了第一个 onCreate，blr x8 进入关键部分。

jnitrace 看一下流程

```
jnitrace -l libjiagu_64.so com.oacia.apk_protect
```

![](https://cdn.jsdelivr.net/gh/1ens/blogImages/202503041950433.png)

```
x8 ====0x7c166e43c0 libart.so!_ZN3art3JNI11GetMethodIDEP7_JNIEnvP7_jclassPKcS6_//JNIEnv->GetMethodID()
x8 ====0x7c164d6a1c libart.so!_ZN3art11ClassLinker15InitializeClassEPNS_6ThreadENS_6HandleINS_6mirror5ClassEEEbb
libart.so! //在类加载过程中执行
libart.so! libart.so! libart.so! bytesToHex
bytesToHex
libart.so!enc2//似乎是m进程检测的关键数据
libart.so!enc2
libart.so!onCreate
libart.so!onCreate
libart.so!Landroid/os/Bundle;
libart.so!Landroid/os/Bundle; 
```

接下来关键操作分析 0x13a61c 函数之后出现了第二个 oncreate，同样是 blr x8, 进入 jni 函数了。

![](https://cdn.jsdelivr.net/gh/1ens/blogImages/202503021017539.png)

```
x8 ====0x7c1671e4d4 libart.so!_ZN3art3JNI15RegisterNativesEP7_JNIEnvP7_jclassPK15JNINativeMethodi//JNIEnv->RegisterNatives()
libart.so!enc2
libart.so!enc2
libart.so!onCreate
libart.so!onCreate
libart.so!Landroid/os/Bundle;
libart.so!Landroid/os/Bundle;
x8 ====0x7c169d0160 libart.so!_ZN3art7Runtime9instance_E
x8 ====0x7c169d0160 libart.so!_ZN3art7Runtime9instance_E
x8 ====0x7c16723374 libart.so!_ZN3art3JNI14ExceptionCheckEP7_JNIEnv// JNIEnv->ExceptionCheck()
```

可以确定，interface11 方法将 onCreate 注册某个 native 方法上

我们去 hook 一下 _ZN3art3JNI15RegisterNativesEP7_JNIEnvP7_jclassPK15JNINativeMethodi 找地址

```
function traceRegisterNatives() {
    var RegisterNativesaddr = null;
    var libartmodule = Process.getModuleByName("libart.so");
    libartmodule.enumerateSymbols().forEach(function (symbol) {
        if (symbol.name == "_ZN3art3JNI15RegisterNativesEP7_JNIEnvP7_jclassPK15JNINativeMethodi") {
            RegisterNativesaddr = symbol.address;
        }
    })
    console.log("RegisterNativesaddr->" + RegisterNativesaddr);
    if (RegisterNativesaddr != null) {
        Interceptor.attach(RegisterNativesaddr, {
            onEnter: function (args) {
                console.log("go into RegisterNativesaddr");
                var methodsptr = ptr(args[2]);
                var method_count = args[3];
                var i = 0;
                console.log("[" + Process.getCurrentThreadId() + "]RegisterNatives->" + method_count);
                for (i = 0; i < method_count; i++) {
                    var name = methodsptr.add(i * Process.pointerSize * 3 + 0).readPointer().readUtf8String();
                    var signature = methodsptr.add(i * Process.pointerSize * 3 + 1 * Process.pointerSize).readPointer().readUtf8String();
                    var fnPtr = methodsptr.add(i * Process.pointerSize * 3 + 2 * Process.pointerSize).readPointer();
                    var find_module = Process.findModuleByAddress(fnPtr);
                    console.log("[" + Process.getCurrentThreadId() + "]RegisterNatives->name:" + name + ",sig:" + signature + ",addr:" + fnPtr);
 
                }
                if (name == "onCreate") {
                    console.log(hexdump(fnPtr));
                    var code = Instruction.parse(fnPtr);
                    var next_code = code.next;
                    console.log(code.address, ":", code);
                    for (var i = 0; i < 10; i++) {
                        var next_c = Instruction.parse(next_code);
                        console.log(next_c.address, ":", next_c);
                        next_code = next_c.next;
                    }
                    var onCreate_fun_ptr = next_c.next.add(0xf);
                    var onCreate_fun_addr = Memory.readPointer(onCreate_fun_ptr);
                 console.log("onCreate_function_address:",onCreate_fun_addr,"offset:",ptr(onCreate_fun_addr));
                }
            }, onLeave: function (retval) {
                 
 
            }
        })
    }
 
}
 
 
function main() {
    var module = Module.findBaseAddress("libjiagu_64.so");
    console.log("libjiagu_64.so base address: " + module);
    traceRegisterNatives();
}
 
setImmediate(main);
```

输出了 oncreate

![](https://cdn.jsdelivr.net/gh/1ens/blogImages/202503041723533.png)

但是是动态注册，找不到 moudel 的名字, 无法直接打印地址。我们直接去 so 里面寻找字节码。在 0x178c50+0xe7000=0x26FC50

![](https://cdn.jsdelivr.net/gh/1ens/blogImages/202503041724006.png)

在 ida 中找到 onCreate 地址

![](https://cdn.jsdelivr.net/gh/1ens/blogImages/202503101749986.png)

定位 vmp 解释器
----------

交叉引用 onCreate，发现他被 sub_1A1908 调用。

再看看 sub_1A1908 的交叉引用

![](https://cdn.jsdelivr.net/gh/1ens/blogImages/202503041818983.png)

来到函数 sub_12DFE4, 这里应该就是函数绑定的地方。

![](https://cdn.jsdelivr.net/gh/1ens/blogImages/202503041819581.png)

参数 0ff_265F20, 指向 sub_137978，进入

![](https://cdn.jsdelivr.net/gh/1ens/blogImages/202503041822558.png)

sub_137978 这个函数是关键操作, 可以 Edit->other->specify switch idiom, 修复。

sub_137978 的逻辑大概是先保存栈帧，然后开辟新的空间，跳转跟之前一样，继续打印 x8 的变化，跟踪函数

```
Call 137978 In
x8 ====0x7164d7d568 libjiagu_64.so!0x139568
x8 ====0x7164d7b9c0 libjiagu_64.so!0x1379c0
x8 ====0x7164d7b9e0 libjiagu_64.so!0x1379e0
x8 ====0x7164d7ba04 libjiagu_64.so!0x137a04
x8 ====0x7164d7ba7c libjiagu_64.so!0x137a7c
x8 ====0x717c4e9ec4 libart.so!_ZN3art3JNI14PushLocalFrameEP7_JNIEnvi
x8 ====0x7164d7bbe8 libjiagu_64.so!0x137be8 //存在异或
x8 ====0x7164d7cd58 libjiagu_64.so!0x138d58
x8 ====0x7164d7cd68 libjiagu_64.so!0x138d68
x8 ====0x7164d7cdf4 libjiagu_64.so!0x138df4
x8 ====0x7164d7cf04 libjiagu_64.so!0x138f04
x8 ====0x7164d7cf0c libjiagu_64.so!0x138f0c
x8 ====0x717c4eb8ec libart.so!_ZN3art3JNI11NewLocalRefEP7_JNIEnvP8_jobject
x8 ====0x7164d7cde8 libjiagu_64.so!0x138de8
x8 ====0x7164d7cdf4 libjiagu_64.so!0x138df4
x8 ====0x7164d7d70c libjiagu_64.so!0x13970c //来到了sub_144EF0
x8 ====0x7164d7d70c libjiagu_64.so!0x13970c //来到了sub_144EF0
x8 ====0x7164d88f4c libjiagu_64.so!0x144f4c //把sj8uwp\}{}r赋值给寄存器
x8 ====0x717c52d374 libart.so!_ZN3art3JNI14ExceptionCheckEP7_JNIEnv
x8 ====0x717c52d374 libart.so!_ZN3art3JNI14ExceptionCheckEP7_JNIEnv
x8 ====0x7164d89018 libjiagu_64.so!0x145018
x8 ====0x7164da3a54 libjiagu_64.so!0x15fa54 //寄存器赋值
x8 ====0x717c4eaa68 libart.so!_ZN3art3JNI12NewGlobalRefEP7_JNIEnvP8_jobject
x8 ====0x7164d89024 libjiagu_64.so!0x145024
x8 ====0x7164d89040 libjiagu_64.so!0x145040
x8 ====0x7164d89058 libjiagu_64.so!0x145058
x8 ====0x7164d890b8 libjiagu_64.so!0x1450b8
x8 ====0x7164d890fc libjiagu_64.so!0x1450fc //存在EOR W8, W8, #31，存入 X0 + 8 位置。把x0的值给了x28
x8 ====0x7164d89148 libjiagu_64.so!0x145148
x8 ====0x7164d9e630 libjiagu_64.so!0x15a630//跳转到sub_16CE0C 存在花指令，似乎是重要函数，调用sub_15FC34
x8 ====0x7164d89154 libjiagu_64.so!0x145154//CMP             W0, #129
x8 ====0x7164d8916c libjiagu_64.so!0x14516c/CMP             W0, #191
x8 ====0x7164d89184 libjiagu_64.so!0x145184//CMP             W0, #224
x8 ====0x7164d89508 libjiagu_64.so!0x145508//来到switch的分发快 CMP      W0, #206
x8 ====0x7164d89520 libjiagu_64.so!0x145520 //CMP             W0, #215
x8 ====0x7164d8a274 libjiagu_64.so!0x146274//CMP             W0, #210
x8 ====0x7164d8a28c libjiagu_64.so!0x14628c//CMP             W0, #212
x8 ====0x7164d8cca4 libjiagu_64.so!0x148ca4  //CMP             W0, #211 很像二分法有没有
x8 ====0x7164d8f3cc libjiagu_64.so!0x14b3cc //来到sub_160D7C解释器，且之前的x28给了x7，也就是参数a8
x8 ====0x717c52d374 libart.so!_ZN3art3JNI14ExceptionCheckEP7_JNIEnv
x8 ====0x717c52d374 libart.so!_ZN3art3JNI14ExceptionCheckEP7_JNIEnv
x8 ====0x717c4eaa68 libart.so!_ZN3art3JNI12NewGlobalRefEP7_JNIEnvP8_jobject
x8 ====0x717c52d374 libart.so!_ZN3art3JNI14ExceptionCheckEP7_JNIEnv
x8 ====0x717c4ee3c0 libart.so!_ZN3art3JNI11GetMethodIDEP7_JNIEnvP7_jclassPKcS6_
x8 ====0x717c52d374 libart.so!_ZN3art3JNI14ExceptionCheckEP7_JNIEnv
x8 ====0x717c52d374 libart.so!_ZN3art3JNI14ExceptionCheckEP7_JNIEnv
libjiagu_64.so!p:}|q
libjiagu_64.so!p:}|q
libjiagu_64.so!p:}|q
libjiagu_64.so!p:}|q
x8 ====0x7164da9f8c libjiagu_64.so!0x165f8c
x8 ====0x7164da5df0 libjiagu_64.so!0x161df0
x8 ====0x717c5093dc libart.so!_ZN3art3JNI25CallNonvirtualVoidMethodAEP7_JNIEnvP8_jobjectP7_jclassP10_jmethodIDP6jvalue
//奔溃
```

定位到我们的解释器 sub_160D7C

定位 opcode 与解密
-------------

使用 ida8.3 进行动态调试

关键从 0x1450fc 处开始 ，从这里开始 hook 会遇到 rtld_db_dlactivity 反调试，而且会引导我们进入错误的分支。

tld_db_dlactivity 函数默认情况下为空函数，当有调试器时将被改为断点指令。Android 反调试对抗中，SIGTRAP 信号反调试可通过判断该函数的数值来判断是否处于被调试状态

我们直接 hook0x15a630，从这里开始调试，我们可以找到加密的 opcode

![](https://cdn.jsdelivr.net/gh/1ens/blogImages/202503121552699.png)

![](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAADIAAAAyCAYAAAAeP4ixAAACbklEQVRoQ+2aMU4dMRCGZw6RC1CSSyQdLZJtKQ2REgoiRIpQkCYClCYpkgIESQFIpIlkW+IIcIC0gUNwiEFGz+hlmbG9b1nesvGW++zxfP7H4/H6IYzkwZFwQAUZmpJVkSeniFJKA8ASIi7MyfkrRPxjrT1JjZ8MLaXUDiJuzwngn2GJaNd7vyP5IoIYY94Q0fEQIKIPRGS8947zSQTRWh8CwLuBgZx479+2BTkHgBdDAgGAC+fcywoyIFWqInWN9BSONbTmFVp/AeA5o+rjKRJ2XwBYRsRXM4ZXgAg2LAPzOCDTJYQx5pSIVlrC3EI45y611osMTHuQUPUiYpiVooerg7TWRwDAlhSM0TuI+BsD0x4kGCuFSRVzSqkfiLiWmY17EALMbCAlMCmI6IwxZo+INgQYEYKBuW5da00PKikjhNNiiPGm01rrbwDwofGehQjjNcv1SZgddALhlJEgwgJFxDNr7acmjFLqCyJuTd6LEGFttpmkYC91Hrk3s1GZFERMmUT01Xv/sQljjPlMRMsxO6WULwnb2D8FEs4j680wScjO5f3vzrlNJszESWq2LYXJgTzjZm56MCHf3zVBxH1r7ftU1splxxKYHEgoUUpTo+grEf303rPH5hxENJqDKQEJtko2q9zGeeycWy3JhpKhWT8+NM/sufIhBwKI+Mta+7pkfxKMtd8Qtdbcx4dUQZcFCQ2I6DcAnLUpf6YMPxhIDDOuxC4C6djoQUE6+tKpewWZ1wlRkq0qUhXptKTlzv93aI3jWmE0Fz2TeujpX73F9TaKy9CeMk8vZusfBnqZ1g5GqyIdJq+XrqNR5AahKr9CCcxGSwAAAABJRU5ErkJggg==)

指向他地址的指针被存在 x0 寄存器之中

![](https://cdn.jsdelivr.net/gh/1ens/blogImages/202503131701638.png)

### 第一部分解密

sub_16CE0C 是第一个解密函数

存在花指令，直接 nop 掉，重新定义为函数，对数据进行异或操作。

![](https://cdn.jsdelivr.net/gh/1ens/blogImages/202503101022965.png)

步入跟踪，来到花指令后的代码

```
loc_16CF9C
ADR             X8, loc_16D09C
SUB             X8, X8, #236
LDR             X30, [SP,#0xA0+var_A0]
MOV             X30, X8
RET
```

**ADR（Address of Label）** 是 ARM64 特有的伪指令，用于**将当前 PC 相对的某个地址加载到寄存器**。

这里，它将 `loc_16D09C` 的地址加载到 `X8`，然后让 X8 的值减少 236（0xEC）

然后从栈中保存 x30 的地址，`X30` 通常用于**存储返回地址**，把 `X8`赋值给 `X30`，ret 时就会返回我们设定的地址

进入真正的解密，解密逻辑为

从加密的 opcode 读取第一字节给 w8，再从存加密 opcode 地址的地方读取第八个字节给 w13 （c5）

w9 存取的是加密的 opcode 读取第二字节，再减去 w12（存加密 opcode 地址的地方读取第八个字节，和 w13 相等），这里结果为 7

赋值，比较，与取`W9`低八位（07），相减，如果 `EQ`（ZF=1，前面 `TST` 结果是 0）`W8 = W11`（0xE2）否则`W8 = W10`（0xE7）

然后异或得 w9=e5，保留 `W13` 低 8 位，来到左边分支，异或得到 w10=6B

![](https://cdn.jsdelivr.net/gh/1ens/blogImages/202503131438694.png)

然后和之前一样的操作来到另外一部分解密操作，w8 恢复到 c5，把 w9 插入到 w10 的高位，然后把 w8 的低位复制到高位

然后 w8 和 w10 异或得到解密的 opcode 20ae

![](https://cdn.jsdelivr.net/gh/1ens/blogImages/202503131442337.png)

### 第二部分解密

把刚刚解密的 opcode，和加密的 opcode 传入函数 sub_15FC34，这里应该是下一步解密，来看看关键部分

x8 赋值为主 dex 文件的地址，w9 是存加密 opcode 地址 + 18 的值（A0AF）, 然后从主 dex 文件加载值 w10 w8

W11 = W9 / W10 （整数除法）  
W9 = W9 - (W11 * W10)（求余数）  
W9 = W9 << 8 （左移 8 位）  
然后 X9 的低 8 位用 X19 的低 8 位替换得到最后值 23AE，得到偏移

最后再从他查询得到值 D2 赋值给 w0

![](https://cdn.jsdelivr.net/gh/1ens/blogImages/202503131457049.png)

接着会根据 w0 的值来进行二分查找。且这个值是会变化的，如果从前面的函数开始调试，会出现错误的值，正确查找来到解释器的位置。

解释器函数入参如下，重点关注 a5 a6（加密的 opcode）和 a7 (已经解密的 opcode), 并且这里的 a4 就是我们之前得到的解密的主 dex 文件地址

```
x0 MOV             W0, #3  ; n4
1 MOV             W1, WZR ; a2
2 MOV             X2, X20 ; JNIEnv
3 MOV             X3, X21 ; a4
4 MOV             X4, X19 ; a5
5 MOV             X5, X22 ; a6
6 MOV             W6, W25 ; a7
7 MOV             X7, X28 ; a8
X0  0000000000000003
X1  0000000000000000
X2  0000007FC4CEE0A0 -> 0000007971A27800 -> 000000797C933118 (libart.so ! _ZN3art3JNI14NewObjectArrayEP7_JNIEnviP7_jclassP8_jobject)
X3  000000797CC43680 -> 0000007962CE6000 (debug003) -> ("dex\n038")
X4  0000007971A26600 ([anon:libc_malloc]) -> 0000000000000000
X5  000000797CD00380 -> 00000079632ACEE0 (debug003) -> EA743718C721CC4E
X6  00000000000020AE
*X7  000000797CCE6C40 -> 0000007FC4CEE0A0 -> 0000007971A27800 -> [...] (libart.so ! _ZN3art3JNI14NewObjectArrayEP7_JNIEnviP7_jclassP8_jobject)
```

### 第三处解密

sub_16CB4C 存在同样的花指令，去除，函数和 sub_16CE0C，套路都是一样的，直接从最后的真实逻辑开始分析

X8 = X20 << 1（2），读取 X19+8 处的字节到 W13（c5），从 X26 + X8 处读取 1 字节到 W8（就是第三个加密的 opcode21）

W9 = W9 - W12（c7-c5=2）, 赋值，测试 W9 的 bit 5 是否为 1，W12 = W9 & 0xFF，W13 = W8 - W13，W8 = (Z == 1) ? W11 : W10

W9 = W8 XOR W12，W8 = W13 & 0xFF，W10 = W8 XOR W11（8E）

![](https://cdn.jsdelivr.net/gh/1ens/blogImages/202503131522837.png)

再来到，和之前的逻辑差不多，最后返回 0x154b=5451

![](https://cdn.jsdelivr.net/gh/1ens/blogImages/202503131531052.png)

而 5451 对应 method_id 正是 onCreate

### 第四处 opcode 解密

sub_16C918 发现也存在异或，分析关键部分

w8 读取第五六位的加密 opcode，（3718）w9 取其高位。

这里 w12 依旧是之前的 c5，四处解密都是用的同一个密钥。

W9 = W9 - W12，W12 = W8 - W12，W8 = W9 & 0xFF，W9 = (Z == 1) ? W11 : W10，W8 = W9 XOR W8

W9 = W12 & 0xFF，后面的逻辑跟前面基本上一样，最后返回 0x0021。

![](https://cdn.jsdelivr.net/gh/1ens/blogImages/202503131544833.png)

总结
--

以上我们得到最后解密的 opcode，逆序一下

ae20（d2）4b152100，这里 d2 是二分查找的关键。

与源 apk 的字节码对比，只有操作数不同。剩下的以此类推。

![](https://cdn.jsdelivr.net/gh/1ens/blogImages/202503131552752.png)

参考：

[360 加固 dex 解密流程分析 | oacia = oacia の BbBlog~ = DEVIL or SWEET](elink@c7bK9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6G2j5h3y4A6j5g2)9J5k6h3c8W2N6W2)9J5c8U0x3$3x3q4)9J5k6r3A6A6j5h3N6#2i4K6u0r3)

[[原创]app 加固分析狗尾续貂之 dex vmp 还原 - Android 安全 - 看雪 - 安全社区 | 安全招聘 | kanxue.com](https://bbs.kanxue.com/thread-285212.htm)

[[分享] 某 vmp 壳原理分析笔记 - Android 安全 - 看雪 - 安全社区 | 安全招聘 | kanxue.com](https://bbs.kanxue.com/thread-225798.htm)

[[原创] 某 DEX_VMP 安全分析与还原 - Android 安全 - 看雪 - 安全社区 | 安全招聘 | kanxue.com](https://bbs.kanxue.com/thread-270799.htm)

[vmp 入门 (一)：android dex vmp 还原和安全性论述](elink@df8K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6E0M7q4)9J5k6i4N6W2K9i4S2A6L8W2)9J5k6i4q4I4i4K6u0W2j5$3!0E0i4K6u0r3M7#2)9J5c8Y4R3$3P5p5W2K9x3$3y4c8z5r3k6K6g2W2u0H3g2@1!0d9b7K6b7#2d9p5p5`.)

[[培训] 内核驱动高级班，冲击 BAT 一流互联网大厂工作，每周日 13:00-18:00 直播授课](https://www.kanxue.com/book-section_list-173.htm)

最后于 2 小时前 被海带编辑 ，原因：