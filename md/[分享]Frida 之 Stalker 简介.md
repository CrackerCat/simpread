> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-265271.htm)

Stalker 是 Frida 的代码跟踪引擎．下面的内容来自 Frida 官网. 在加密解密，trace，代码定位方面用途还是挺大的．

> Stalker is Frida’s code tracing engine. It allows threads to be followed, capturing every function, every block, even every instruction which is executed

类似 Frida 的工具还有 QBDI.

### QBDI

> QuarkslaB Dynamic binary Instrumentation (QBDI) is a modular, cross-platform and cross-architecture DBI framework. It aims to support Linux, macOS, Android, iOS and Windows operating systems running on x86, x86-64, ARM and AArch64 architectures.

QBDI 也能够和 Frida 完美结合．举个例子，打印导出函数 aFunction 对应的汇编代码．

```
var vm = new QBDI();
var state = vm.getGPRState();
vm.allocateVirtualStack(state, 0x1000000);
var funcPtr = Module.findExportByName(null, "aFunction");
vm.addInstrumentedModuleFromAddr(funcPtr);
var icbk = vm.newInstCallback(function(vm, gpr, fpr, data) {
    var inst = vm.getInstAnalysis();
    // Display instruction dissassembly
    fmt = "0x" + inst.address.toString(16) + " " + inst.disassembly;
    console.log(fmt);
    return VMAction.CONTINUE;
});
vm.addCodeCB(InstPosition.PREINST, icbk);
vm.call(funcPtr, [42]);


```

QBDI 在 Android 平台下使用需要 root 以及关闭 SELinux.

```
host$ adb root
host$ adb shell setenforce 0


```

### 参考

https://frida.re/news/2019/12/18/frida-12-8-released/ https://bbs.pediy.com/thread-264680.htm https://frida.re/docs/stalker/ [Frida 工作原理](https://mp.weixin.qq.com/s?__biz=MzI1NjEyMzgxMw==&mid=2247484822&idx=1&sn=82bd128162c632d9956438c6233ead60&chksm=ea2a335cdd5dba4a0652bba48fc4c9cbfb8cad34d3266cffc41f94eadf091060d8e298547958&token=313219613&lang=zh_CN#rd "Frida工作原理") https://github.com/NMHai/frida-qbdi-fuzzer

### Stalker 实现

### 基本原理

Stalker 是基于**动态重新编译**的代码跟踪器。 它将代码指令复制到内存中的另一个位置，在该位置对其进行调整以适应新位置并包括其他跟踪指令。 **如果应用程序在原始位置检查其代码，则会发现该代码是完整无缺的**，因为它是被篡改的代码的副本。

下图显示了函数如何在其新的内存位置进行 **transform** 以包括代码跟踪.![](https://img-blog.csdnimg.cn/20210114111820374.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2hlbGxvd29ybGRkbQ==,size_16,color_FFFFFF,t_70#pic_center)

### 练习一下

a.out 程序源码：

```
#include<stdio.h>
#include<unistd.h>
static int fn(int n){
    printf("number is %d\n",n);
}

int main(int argc, char ** argv){
    int n = 0;
    printf("fn is at %p\n",&fn);
    while(1){
        fn(n++);
        sleep(3);
    }
    return 0;
}



```

Frida 脚本 (重点在 transform):

```
var app = new ModuleMap(isAppAddress);

Process.enumerateThreadsSync().forEach(function(thread){
    Stalker.follow(thread.id,{
        transform: function (iterator) {
            var instruction = iterator.next();
            if (!app.has(instruction.address)) {
                do{
                    iterator.keep();
                }  while ((iterator.next()) !== null);
                return ;
            }

            do{
                console.log(instruction.address + ":" + instruction);
                iterator.keep();
            }  while ((instruction = iterator.next()) !== null);

        }
    })
})

function isAppAddress(module) {
    return module.path.indexOf("a.out") != -1;
}


```

运行结果:![](https://img-blog.csdnimg.cn/20210114115752391.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2hlbGxvd29ybGRkbQ==,size_16,color_FFFFFF,t_70#pic_center) 关闭系统的 ASLR(地址随机化)，对比指令和地址，显然是对应的。**基于此我们可以定位到 fn 函数被执行了。**![](https://img-blog.csdnimg.cn/20210114120004826.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2hlbGxvd29ybGRkbQ==,size_16,color_FFFFFF,t_70#pic_center)如果能够实施跟踪寄存器的值就更好了，这个也很容易实现，只需要添加下面的代码就可以了。**基于此便可以实现类似 IDA 的 trace 功能。**

```
 iterator.putCallout((context) => {
   console.log(JSON.stringify(context))
 })


```

### stalker 做了什么

从使用者的角度，每当执行一个基本快，stalker 都会做以下事情:

1. 对于方法调用，保存 lr 等必要信息 2. 重定位位置相关指令，例如：ADR Xd, label3. 建立此块的索引，如果此块在达到可信阈值后，内容未曾变化，下次将不再重新编译（为了加快速度）4. 根据 transform 函数，编译生成一个新的基本块 GumExecBlock ，保存到 GumSlab 。5. 在上一过程中，可通过 void transform(GumStalkerIterator iterator, GumStalkerOutput output, gpointer user_data) 来读取，改动，写入指令。6.transform 过程中可通过 void gum_stalker_iterator_put_callout (GumStalkerIterator self,GumStalkerCallout callout, gpointer data, GDestroyNotify data_destroy) 来设置一个当此位置被执行到时的 callout。通过此 void callout(GumCpuContext cpu_context, gpointer user_data) 获取 cpu 信息。7. 执行一个基本快 GumExecBlock，开始下一个基本快

### trap 功能

sleep 函数定义:

> 相关函数：signal, alarm 头文件：#include <unistd.h> 定义函数：unsigned int sleep(unsigned int seconds); 函数说明：sleep() 会令目前的进程暂停, 直到达到参数 seconds 所指定的时间, 或是被信号所中断. 返回值：若进程 / 线程挂起到参数所指定的时间则返回 0，若有信号中断则返回剩余秒数。

这里 trap sleep 函数， **step into** 的时候不正是重新进入一个函数吗.

```
var sleep = new NativeFunction(
  Module.getExportByName(null, 'sleep'),
  'uint', ['int'],
  { traps: 'all' }
);

Stalker.follow({
events: {
  call: true
},
onReceive: function (e) {
  console.log(JSON.stringify(Stalker.parse(e)));
}
});

var result = sleep(4);
console.log('sleep', result);


```

By setting the traps: 'all' option on the NativeFunction, it will re-activate Stalker when called from a thread where Stalker is temporarily paused because it’s calling out to an excluded range – which is the case here because all of frida-agent’s code is marked as excluded.

### stalker 实践

### 定位函数位置

### Thread.backtrace

同样是上面的 a.out，可以通过如下的方式来定位 fn 的地址。

```
Interceptor.attach(target_printf, {
    onEnter: function (args) {
        console.log('printf', args[0]);
        console.log(Thread.backtrace(this.context, Backtracer.ACCURATE).map(DebugSymbol.fromAddress).join('\n') + '\n');
    }
});


```

### 遍历 symbols

也可以通过遍历来定位 fn 的地址

```
for(var i = 0; i < symbols.length; i++) {
    targetFun =  ptr(symbols[i].address);
    if (symbols[i].name == "fn" && symbols[i].type == "function"){
        console.log("enter fn" + symbols[i].size);
        console.log("start:" + symbols[i].address);
        //下面的代码可以实现根据地址定位模块
       //let moduleMap = new ModuleMap();    
       // let targetModule = moduleMap.find(symbols[i].address);
       //console.log("target module" + JSON.stringify(targetModule));
  }
}


```

### 写在最后

在接下来的文章中，会详细介绍 Frida 中 Stalker 的工作原理以及 C 模块。同时基于 Frida 的 Stalker 进行软件的破解。

有兴趣可以比较下下面这两段代码是否有区别。

```
 do{
   if (isModule){
     console.log(instruction.address + ":" + instruction);
   }
   iterator.keep();
} while ((instruction = iterator.next()) !== null);


```

和

```
if (!isModule) {
    do{
        iterator.keep();
    }  while ((iterator.next()) !== null);
    return ;
}

do{
    console.log(instruction.address + ":" + instruction);
    iterator.keep();
}  while ((instruction = iterator.next()) !== null);


```

### 公众号

更多 Frida 相关的内容，欢迎关注我的微信公众号: 无情剑客。![](https://img-blog.csdnimg.cn/20210114123135143.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2hlbGxvd29ybGRkbQ==,size_16,color_FFFFFF,t_70#pic_center)

[[招聘] 欢迎你加入看雪团队！](https://job.kanxue.com/company-read-31.htm)