> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-267499.htm)

虚假分支的混淆会在增加大量的 if else 分支。增加静态分析的复杂度。但是实际在动态执行的时候。很多 if else 实际都是没有执行的。所以去掉虚假分支其实就是删除掉那些没有执行到的代码块。那么我们只要知道目标函数中，哪些汇编代码执行了，并且记录下执行汇编的 address。然后把这些汇编以外的代码全部标记为 nop。然后再用 ida 反汇编看到的结果。就直接是去掉虚假分支的结果了。

 

下面我们从准备环境开始。

[](#1、编译ollvm)1、编译 ollvm
------------------------

​ `mkdir ~/ollvm/llvm-project-llvmorg-9.0.1/build-release`

 

​ `cd ~/ollvm/llvm-project-llvmorg-9.0.1/build-release`

 

​ `cmake -G Ninja -DCMAKE_BUILD_TYPE=Release -DLLVM_ENABLE_PROJECTS="clang;clang++" ../llvm`

 

​ `ninja`

[](#2、配置ndk，使用ollvm的clang来进行编译apk)2、配置 ndk，使用 ollvm 的 clang 来进行编译 apk
---------------------------------------------------------------------

​ 首先下载 ndk，地址：https://developer.android.google.cn/ndk/downloads/

 

​ 然后将 ollvm 编译好的 cmake-build-release 这个目录下的 bin、lib、include 三个目录拷贝到下载的 ndk 的目录~/ndk/toolchains/llvm/prebuilt/linux-x86_64 / 中。

 

​ 然后修改我们的 apk 项目中的使用 ndk 的目录。否则会默认使用 sdk 中的 ndk 来编译。找到 local.properties 增加`ndk.dir=/home/king/android-ndk-r21e`

 

​ 最后配置 ollvm 的参数在 cpp/CMakeLists.txt 中添加配置`add_definitions("-mllvm -bcf -mllvm -bcf_loop=3")`

 

​ 然后这里碰到了一个问题。就是带上 - mllvm -bcf 这个参数。就会导致一直在 gradle build 中。这个实际上是 ollvm 的一个 bug。由于我拉去的分支是没有修复这个 bug 的。所以需要根据这个地址的提交代码修改下就可以了 https://github.com/obfuscator-llvm/obfuscator/pull/76

[](#3、写个native-cpp的简单逻辑代码。用来作为反混淆的目标。)3、写个 native-cpp 的简单逻辑代码。用来作为反混淆的目标。
-------------------------------------------------------------------------

```
extern "C" JNIEXPORT jstring JNICALL
Java_com_example_ollvmdemo2_MainActivity_stringFromJNI(
        JNIEnv *env,
        jobject /* this */) {
 
    std::string hello = "Hello from C++";
    if (hello.length()>10){
        hello="ceshi";
    }else if(hello.length()>30){
        hello="ollvm";
    }else{
        hello="fla";
    }
    return env->NewStringUTF(hello.c_str());
}

```

编译后。使用 ida 打开 native-lib.so。然后发现混淆没有起作用

 

![](https://bbs.pediy.com/upload/attach/202105/659397_EX23KYRTXEC25ZA.png)

 

这里编译使用的是 clang++ 编译的。所以我们用 clang++ 简单的测试一下混淆

```
#include int main(int argc,const char** argv) {
    std::cout << "Hello, World!" << std::endl;
    std::string hello="heheda ollvm";
    if (hello.length()>10){
        hello="ceshi";
    }else if(hello.length()>30){
        hello="ollvm";
    }else{
        hello="fla";
    }
    std::cout << hello.c_str() << std::endl;
    return 0;
} 
```

然后在 ollvm 的混淆处手动打个日志。看看混淆的目标函数是什么

```
void bogus(Function &F) {
    errs() << "bcf: Started on function " << F.getName() << "\n";
    ...
    ...
}

```

接着编译一下我们的测试例子

 

`clang++ -mllvm -bcf main.cpp`

 

编译结果如下，并没有看到对我们的 main 函数进行混淆处理。然后我们再测试下 clang 的。

```
bcf: Started on function __cxx_global_var_init
bcf: Started on function _GLOBAL__sub_I_main.cpp

```

`clang -mllvm -bcf main.c`

 

编译结果如下，有对我们的 main 函数进行混淆

```
bcf: Started on function main

```

那么可以看出区别了。当我们是对 c++ 的项目进行混淆时。要避免需要保护的代码在主函数中。因此，我们再修改下上面的测试代码

```
#include std::string calcKey(std::string data){
    if (data.length()>10){
        data="ceshi";
    }else if(data.length()>30){
        data="ollvm";
    }else{
        data="fla";
    }
    return data;
}
extern "C" JNIEXPORT jstring JNICALL
Java_com_example_ollvmdemo2_MainActivity_stringFromJNI(
        JNIEnv *env,
        jobject /* this */) {
 
    std::string hello = "Hello from C++";
    hello=calcKey(hello);
    return env->NewStringUTF(hello.c_str());
} 
```

那么再进行一次混淆。结果如下。

 

![](https://bbs.pediy.com/upload/attach/202105/659397_CD9HA9K8Y229UUB.png)

 

我们再看看 calcKey 里面想要保护的代码是否有混淆

 

![](https://bbs.pediy.com/upload/attach/202105/659397_3FRMUR9QU3Q5NDZ.png)

 

到这里准备的案例程序 ok 了。接着我们开始还原这个虚假分支的混淆。首先我们需要知道这个函数中哪些汇编执行了。所以我们要 trace 打印每一句执行了的汇编代码地址。

[](#4、使用unidbg模拟执行so)4、使用 unidbg 模拟执行 so
----------------------------------------

先贴上大佬的 github 地址: https://github.com/zhkl0228/unidbg

 

然后先用 unidbg 把我们的 so 给模拟执行起来。下面贴上代码

```
public class BcfTest {
private final AndroidEmulator emulator;
private final VM vm;
private final DvmClass mainActivityDvm;
public static void main(String[] args) {
  BcfTest bcfTest = new BcfTest();
  bcfTest.call_calckey();
}
private BcfTest(){
  emulator = AndroidEmulatorBuilder
    .for64Bit()
    .build();
  Memory memory = emulator.getMemory();
  LibraryResolver resolver = new AndroidResolver(23);
  memory.setLibraryResolver(resolver);
  vm = emulator.createDalvikVM(null);
  vm.setVerbose(false);
  mainActivityDvm = vm.resolveClass("com/example/ollvmdemo2/MainActivity");
  DalvikModule dm = vm.loadLibrary(new File("unidbg-android/src/test/resources/example_binaries/ollvm_bcf/libnative-lib.so"), false);
  dm.callJNI_OnLoad(emulator);
}
//主动调用目标函数
private void call_calckey(){
  //调用一个返回值为object的静态的jni函数
  StringObject res = mainActivityDvm.callStaticJniMethodObject(emulator, "stringFromJNI()Ljava/lang/String;");
  System.out.println(res.toString());
}

```

执行结果是直接显示`ceshi`。接着我们需要把这个函数执行过程中的每个执行的汇编地址打印一下。

 

首先找到 calcKey 函数的起始位置 0x124CC 以及函数大小 0x838

 

![](https://bbs.pediy.com/upload/attach/202105/659397_6KTWZ5CA4F4HZGX.png)

 

接着给我们之前的代码加上一个 tracecode 监控。直接修改那个调用函数。在调用前先挂上监控

```
private void call_calckey(){
        emulator.getBackend().hook_add_new(new CodeHook() {
            @Override
            public void hook(Backend backend, long address, int size, Object user) {
                  //打印当前地址。这里要把unidbg使用的基址给去掉。
                System.out.println(String.format("0x%x",address-0x40000000));
            }
        },0x400124CC,0x400124CC+0x838,null);
        //调用一个返回值为object的静态的jni函数
        StringObject res = mainActivityDvm.callStaticJniMethodObject(emulator, "stringFromJNI()Ljava/lang/String;");
        System.out.println(res.toString());
    }

```

执行完成后得到如下结果

```
0x124cc
0x124d0
0x124d4
0x124d8
0x124dc
0x124e0
0x124e4
0x124e8
0x124ec
0x124f0
0x124f4
0x124f8
0x124fc
0x12500
0x12504
0x12508
0x1250c
0x12510
0x12514
0x12518
0x1251c
0x12520
0x12524
0x12528
0x1252c
0x12530
0x12534
0x12538
0x12540
0x12544
0x12548
0x1254c
0x12550
0x12554
0x12558
0x1255c
0x12560
0x12564
0x12568
0x12d04
0x1256c
0x12570
0x12574
0x12578
0x1257c
0x12580
0x12584
0x12588
0x1258c
0x12590
0x12594
0x12598
0x1259c
0x125a0
0x125a4
0x125a8
0x125ac
0x125b4
0x125b8
0x125bc
0x125c0
0x125c4
0x125c8
0x125cc
0x125d0
0x12b54
0x12b58
0x12b5c
0x12b60
0x12b64
0x12b68

```

这里的汇编地址就都是确定有执行到的了。

[](#5、idapython去虚假指令)5、idapython 去虚假指令
--------------------------------------

上面获取到执行了的汇编地址。接下来我们用 ida 来执行 py 脚本把执行的部分进行高亮。并且将这个函数范围中的未执行的部分代码修改成 nop。改成 nop 之后 ida 就不会把未执行的汇编解析出来了。这里可以直接使用 idapython，也可以用对 idapython 进行包装的库。这里的例子用的 sark 来进行处理。https://github.com/tmr232/Sark

 

用 ida 的插件 keypatch 看了下 nop 的对应字节是 1f 20 03 d5

 

![](https://bbs.pediy.com/upload/attach/202105/659397_SDD9Y5EQ6MC8N7J.png)

 

下面附上 py 代码

```
# -*- coding: utf-8 -*-
import sark
import idc
import sys
def patch_code(addr,code):
    for i in range(len(code)):
        idc.patch_byte(addr+i,code[i])
 
#将指定地址的字节修改成nop
def nop(addr):
    nop_code=[0x1f,0x20,0x03,0xd5]
    patch_code(addr,nop_code)
 
def main():
    logfile
    #定义要处理函数的起始和终止范围
    start=0x124CC
    end=0x124CC+0x838
    addrs=[]
    #将所有执行的汇编地址读取进来
    with open(logfilename,"r") as logfile:
        lines=logfile.readlines()
        for line in lines:
            addrs.append(int(line.replace("\n",""),16))
    #所有执行过的汇编地址修改一下颜色。高亮起来便于查看
    for addr in addrs:
        line=sark.line.Line(addr)
        line.color=0x00ffff
 
    #获取到目标函数内的所有汇编地址
    funcLines=sark.lines(start,end)
    for line in funcLines:
        #如果该行是code则判断颜色是否被我们标注成有效代码。
        if line.type=="code":
            if line.color!=0x00ffff:
                nop(line.ea)
 
 
if __name__=="__main__":
    main()

```

最后 ida 执行脚本一下。下面附上处理后的效果图

 

![](https://bbs.pediy.com/upload/attach/202105/659397_55WSS5JUZQ35CYF.png)

 

然后看看 F5 解析出来的变成什么样子了

 

![](https://bbs.pediy.com/upload/attach/202105/659397_N6QXKAD5NWUE3V2.png)

[[看雪官方培训] Unicorn Trace 还原 Ollvm 算法！《安卓高级研修班》2021 年 6 月班火热招生！！](https://bbs.pediy.com/thread-267018.htm)