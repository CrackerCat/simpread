> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.anquanke.com](https://www.anquanke.com/post/id/195215)

[![](https://p1.ssl.qhimg.com/t0167bf11e82f16975b.png)](https://p1.ssl.qhimg.com/t0167bf11e82f16975b.png)

前言
--

大家好，窝又来写文章了，咱们现在在这篇文章中，我们来对其官方的一些非常常用的`API`进行学习。所谓工欲善其事，必先利其器。想要好好学习`FRIDA`我们就必须对`FRIDA API`深入的学习以对其有更深的了解和使用，通常大部分核心原理也在官方`API`中写着，我们学会来使用一些案例来结合`API`的使用。

注意，运行以下任何代码时都需要提前启动手机中的`frida-server`文件。

系列文章目录搬新 “家” 了，地址：[https://github.com/r0ysue/AndroidSecurityStudy](https://github.com/r0ysue/AndroidSecurityStudy) ，接下来窝会努力写更多喔 ~

1.1 FRIDA 输出打印
--------------

### 1.1.1 console 输出

不论是什么语言都好，第一个要学习总是如何输出和打印，那我们就来学习在`FRIDA`打印值。在官方 API 有两种打印的方式，分别是`console`、`send`，我们先来学习非常的简单的`console`，这里我创建一个`js`文件，代码示例如下。

```
function hello_printf() {
    Java.perform(function () {
        console.log("");
        console.log("hello-log");
        console.warn("hello-warn");
        console.error("hello-error");
    });
}
setImmediate(hello_printf,0);


```

当文件创建好之后，我们需要运行在手机中安装的`frida-server`文件，在上一章我们学过了如何安装在`android`手机安装`frida-server`，现在来使用它，我们在`ubuntu`中开启一个终端，运行以下代码，启动我们安装好的`frida-server`文件。

```
roysue@ubuntu:~$ adb shell
sailfish:/ $ su
sailfish:/ $ ./data/local/tmp/frida-server


```

然后执行以下代码，对目标应用`app`的进程`com.roysue.roysueapplication`使用`-l`命令注入`Chap03.js`中的代码`1-1`以及执行脚本之后的效果图`1-1`！

`frida -U com.roysue.roysueapplication -l Chap03.js`

代码 1-1 代码示例

[![](https://p2.ssl.qhimg.com/t01fefc9bc3703790ae.png)](https://p2.ssl.qhimg.com/t01fefc9bc3703790ae.png)

图 1-1 终端执行

可以到终点已经成功注入了脚本并且打印了`hello`，但是颜色不同，这是`log`的级别的原因，在`FRIDA`的`console`中有三个级别分别是`log、warn、error`。

<table><thead><tr><th>级别</th><th>含义</th></tr></thead><tbody><tr><td>log</td><td>正常</td></tr><tr><td>warn</td><td>警告</td></tr><tr><td>error</td><td>错误</td></tr></tbody></table>

### 1.1.2 console 之 hexdump

`error`级别最为严重其`次warn`，但是一般在使用中我们只会使用`log`来输出想看的值；然后我们继续学习`console`的好兄弟，`hexdump`，其含义: 打印内存中的地址，`target`参数可以是`ArrayBuffer`或者`NativePointer`, 而`options`参数则是自定义输出格式可以填这几个参数`offset、lengt、header、ansi`。

`hexdump`代码示例以及执行效果如下。

```
var libc = Module.findBaseAddress('libc.so');
console.log(hexdump(libc, {
  offset: 0,
  length: 64,
  header: true,
  ansi: true
}));
           0  1  2  3  4  5  6  7  8  9  A  B  C  D  E  F  0123456789ABCDEF
00000000  7f 45 4c 46 01 01 01 00 00 00 00 00 00 00 00 00  .ELF............
00000010  03 00 28 00 01 00 00 00 00 00 00 00 34 00 00 00  ..(.........4...
00000020  34 a8 04 00 00 00 00 05 34 00 20 00 08 00 28 00  4.......4. ...(.
00000030  1e 00 1d 00 06 00 00 00 34 00 00 00 34 00 00 00  ........4...4...


```

### **1.1.3 send**

`send`是在`python`层定义的`on_message`回调函数，`jscode`内所有的信息都被监控`script.on('message', on_message)`，当输出信息的时候`on_message`函数会拿到其数据再通过`format`转换， 其最重要的功能也是最核心的是能够直接将数据以`json`格式输出，当然数据是二进制的时候也依然是可以使用`send`，十分方便，我们来看代码`1-2`示例以及执行效果。

```
import frida
import sys

def on_message(message, data):
    if message['type'] == 'send':
        print("[*] {0}".format(message['payload']))
    else:
        print(message)

jscode = """
    Java.perform(function () 
    {
        var jni_env = Java.vm.getEnv();
        console.log(jni_env);
        send(jni_env);
    });
 """

process = frida.get_usb_device().attach('com.roysue.roysueapplication')
script = process.create_script(jscode)
script.on('message', on_message)
script.load()
sys.stdin.read()

运行脚本效果如下：

roysue@ubuntu:~/Desktop/Chap09$ python Chap03.py 
[object Object]
[*] {'handle': '0xdf4f8000', 'vm': {}}


```

可以看出这里两种方式输出的不同的效果，`console`直接输出了`[object Object]`，无法输出其正常的内容，因为`jni_env`实际上是一个对象，但是使用`send`的时候会自动将对象转`json`格式输出。通过对比，我们就知道`send`的好处啦~

1.2 FRIDA 变量类型
--------------

学完输出之后我们来学习如何声明变量类型。

<table><thead><tr><th>索引</th><th>API</th><th>含义</th></tr></thead><tbody><tr><td>1</td><td>new Int64(v)</td><td>定义一个有符号 Int64 类型的变量值为 v，参数 v 可以是字符串或者以 0x 开头的的十六进制值</td></tr><tr><td>2</td><td>new UInt64(v)</td><td>定义一个无符号 Int64 类型的变量值为 v，参数 v 可以是字符串或者以 0x 开头的的十六进制值</td></tr><tr><td>3</td><td>new NativePointer(s)</td><td>定义一个指针，指针地址为 s</td></tr><tr><td>4</td><td>ptr(“0”)</td><td>同上</td></tr></tbody></table>

代码示例以及效果

```
Java.perform(function () {
    console.log("");
    console.log("new Int64(1):"+new Int64(1));
    console.log("new UInt64(1):"+new UInt64(1));
    console.log("new NativePointer(0xEC644071):"+new NativePointer(0xEC644071));
    console.log("new ptr('0xEC644071'):"+new ptr(0xEC644071));
});
    输出效果如下：
    new Int64(1):1
    new UInt64(1):1
    new NativePointer(0xEC644071):0xec644071
    new ptr('0xEC644071'):0xec644071


```

`frida`也为`Int64(v)`提供了一些相关的 API：

<table><thead><tr><th>索引</th><th>API</th><th>含义</th></tr></thead><tbody><tr><td>1</td><td>add(rhs)、sub(rhs)、and(rhs)、or(rhs)、xor(rhs)</td><td>加、减、逻辑运算</td></tr><tr><td>2</td><td>shr(N)、shl(n)</td><td>向右 / 向左移位 n 位生成新的 Int64</td></tr><tr><td>3</td><td>Compare(Rhs)</td><td>返回整数比较结果</td></tr><tr><td>4</td><td>toNumber()</td><td>转换为数字</td></tr><tr><td>5</td><td>toString([radix=10])</td><td>转换为可选基数的字符串 (默认为 10)</td></tr></tbody></table>

我也写了一些使用案例，代码如下。

```
function hello_type() {
    Java.perform(function () {
        console.log("");
        
        console.log("8888 + 1:"+new Int64("8888").add(1));
        
        console.log("8888 - 1:"+new Int64("8888").sub(1));
        
        console.log("8888 << 1:"+new Int64("8888").shr(1));
        
        console.log("8888 == 22:"+new Int64("8888").compare(22));
        
        console.log("8888 toString:"+new Int64("8888").toString());
    });
}


```

代码执行效果如图 1-2。

[![](https://p4.ssl.qhimg.com/t01110638f85fa6a01f.png)](https://p4.ssl.qhimg.com/t01110638f85fa6a01f.png)

图 1-2 Int64 API

1.3 RPC 远程调用
------------

可以替换或插入的空对象，以向应用程序公开`RPC`样式的`API`。该键指定方法名称，该值是导出的函数。此函数可以返回一个纯值以立即返回给调用方，或者承诺异步返回。也就是说可以通过 rpc 的导出的功能使用在`python`层，使`python`层与`js`交互，官方示例代码有`Node.js`版本与`python`版本，我们在这里使用`python`版本，代码如下。

### 1.3.1 远程调用代码示例

```
import frida

def on_message(message, data):
    if message['type'] == 'send':
        print(message['payload'])
    elif message['type'] == 'error':
        print(message['stack'])

session = frida.get_usb_device().attach('com.roysue.roysueapplication')

source = """
    rpc.exports = {
    add: function (a, b) {
        return a + b;
    },
    sub: function (a, b) {
        return new Promise(function (resolve) {
        setTimeout(function () {
            resolve(a - b);
        }, 100);
        });
    }
    };
"""

script = session.create_script(source)
script.on('message', on_message)
script.load()
print(script.exports.add(2, 3))
print(script.exports.sub(5, 3))
session.detach()


```

### 1.3.2 远程调用代码示例详解

官方源码示例是附加在目标进程为`iTunes`，再通过将`rpc`的`./agent.js`文件读取到`source`，进行使用。我这里修改了附加的目标的进程以及直接将`rpc`的代码定义在`source`中。我们来看看这段是咋运行的，仍然先对目标进程附加，然后在写`js`中代码，也是`source`变量，通过`rpc.exports`关键字定义需要导出的两个函数，上面定义了`add`函数和`sub`函数，两个的函数写作方式不一样，大家以后写按照`add`方法写就好了，`sub`稍微有点复杂。声明完函数之后创建了一个脚本并且注入进程，加载了脚本之后可以到`print(script.exports.add(2, 3))`以及`print(script.exports.sub(5, 3))，`在`python`层直接调用。`add`的返回的结果为`5`，`sub`则是`2`，下见下图`1-3`。

[![](https://p5.ssl.qhimg.com/t01637fc0ee367df7b2.png)](https://p5.ssl.qhimg.com/t01637fc0ee367df7b2.png)

图 1-3 执行 python 脚本

1.4 Process 对象
--------------

我们现在来介绍以及使用一些`Process`对象中比较常用的`api`~

### 1.4.1 Process.id

`Process.id`：返回附加目标进程的`PID`

### 1.4.2 Process.isDebuggerAttached()

`Process.isDebuggerAttached()`：检测当前是否对目标程序已经附加

### 1.4.3 Process.enumerateModules()

枚举当前加载的模块，返回模块对象的数组。  
`Process.enumerateModules()`会枚举当前所有已加载的`so`模块，并且返回了数组`Module`对象，`Module`对象下一节我们来详细说，在这里我们暂时只使用`Module`对象的`name`属性。

```
function frida_Process() {
    Java.perform(function () {
        var process_Obj_Module_Arr = Process.enumerateModules();
        for(var i = 0; i < process_Obj_Module_Arr.length; i++) {
            console.log("",process_Obj_Module_Arr[i].name);
        }
    });
}
setImmediate(frida_Process,0);


```

我来们开看看这段`js`代码写了啥：在`js`中能够直接使用`Process`对象的所有`api`，调用了`Process.enumerateModules()`方法之后会返回一个数组，数组中存储 N 个叫 Module 的对象，既然已经知道返回了的是一个数组，很简单我们就来`for`循环它便是，这里我使用下标的方式调用了`Module`对象的`name`属性，`name`是`so`模块的名称。见下图`1-4`。

[![](https://p3.ssl.qhimg.com/t011b426bba83da9bb4.png)](https://p3.ssl.qhimg.com/t011b426bba83da9bb4.png)

图 1-4 终端输出了所有已加载的 so

### 1.4.4 Process.enumerateThreads()

`Process.enumerateThreads()`：枚举当前所有的线程，返回包含以下属性的对象数组：

<table><thead><tr><th>索引</th><th>属性</th><th>含义</th></tr></thead><tbody><tr><td>1</td><td>id</td><td>线程 id</td></tr><tr><td>2</td><td>state</td><td>当前运行状态有 running, stopped, waiting, uninterruptible or halted</td></tr><tr><td>3</td><td>context</td><td>带有键 pc 和 sp 的对象，它们是分别为 ia32/x64/arm 指定 EIP/RIP/PC 和 ESP/RSP/SP 的 NativePointer 对象。也可以使用其他处理器特定的密钥，例如 eax、rax、r0、x0 等。</td></tr></tbody></table>

使用代码示例如下：

```
function frida_Process() {
    Java.perform(function () {
       var enumerateThreads =  Process.enumerateThreads();
       for(var i = 0; i < enumerateThreads.length; i++) {
        console.log("");
        console.log("id:",enumerateThreads[i].id);
        console.log("state:",enumerateThreads[i].state);
        console.log("context:",JSON.stringify(enumerateThreads[i].context));
        }
    });
}
setImmediate(frida_Process,0);


```

获取当前是所有线程之后返回了一个数组，然后循环输出它的值，如下图`1-5`。

[![](https://p1.ssl.qhimg.com/t015d7ed42cbf128976.png)](https://p1.ssl.qhimg.com/t015d7ed42cbf128976.png)

图 1-4 终端执行

### 1.4.5 Process.getCurrentThreadId()

`Process.getCurrentThreadId()`：获取此线程的操作系统特定 `ID` 作为数字

1.5 Module 对象
-------------

3.4 章节中`Process.EnumererateModules()`方法返回了就是一个`Module`对象，咱们这里来详细说说`Module`对象，先来瞧瞧它都有哪些属性。

### 1.5.1 Module 对象的属性

<table><thead><tr><th>索引</th><th>属性</th><th>含义</th></tr></thead><tbody><tr><td>1</td><td>name</td><td>模块名称</td></tr><tr><td>2</td><td>base</td><td>模块地址，其变量类型为 NativePointer</td></tr><tr><td>3</td><td>size</td><td>大小</td></tr><tr><td>4</td><td>path</td><td>完整文件系统路径</td></tr></tbody></table>

除了属性我们再来看看它有什么方法。

### 1.5.2 Module 对象的 API

<table><thead><tr><th>索引</th><th>API</th><th>含义</th></tr></thead><tbody><tr><td>1</td><td>Module.load()</td><td>加载指定 so 文件，返回一个 Module 对象</td></tr><tr><td>2</td><td>enumerateImports()</td><td>枚举所有 Import 库函数，返回 Module 数组对象</td></tr><tr><td>3</td><td>enumerateExports()</td><td>枚举所有 Export 库函数，返回 Module 数组对象</td></tr><tr><td>4</td><td>enumerateSymbols()</td><td>枚举所有 Symbol 库函数，返回 Module 数组对象</td></tr><tr><td>5</td><td>Module.findExportByName(exportName)、Module.getExportByName(exportName)</td><td>寻找指定 so 中 export 库中的函数地址</td></tr><tr><td>6</td><td>Module.findBaseAddress(name)、Module.getBaseAddress(name)</td><td>返回 so 的基地址</td></tr></tbody></table>

### 1.5.3 Module.load()

在`frida-12-5`版本中更新了该`API`，主要用于加载指定`so`文件，返回一个`Module`对象。

使用代码示例如下：

```
function frida_Module() {
    Java.perform(function () {
         
         const hooks = Module.load('libhello.so');
         
         console.log("模块名称:",hooks.name);
         console.log("模块地址:",hooks.base);
         console.log("大小:",hooks.size);
         console.log("文件系统路径",hooks.path);
    });
}
setImmediate(frida_Module,0);

输出如下：
模块名称: libhello.so
模块地址: 0xdf2d3000
大小: 24576
文件系统路径 /data/app/com.roysue.roysueapplication-7adQZoYIyp5t3G5Ef5wevQ==/lib/arm/libhello.so


```

### 1.5.4 Process.EnumererateModules()

咱们这一小章节就来使用`Module`对象，把上章的`Process.EnumererateModules()`对象输出给它补全了，代码如下。

```
function frida_Module() {
    Java.perform(function () {

        var process_Obj_Module_Arr = Process.enumerateModules();
        for(var i = 0; i < process_Obj_Module_Arr.length; i++) {
            if(process_Obj_Module_Arr[i].path.indexOf("hello")!=-1)
            {
                console.log("模块名称:",process_Obj_Module_Arr[i].name);
                console.log("模块地址:",process_Obj_Module_Arr[i].base);
                console.log("大小:",process_Obj_Module_Arr[i].size);
                console.log("文件系统路径",process_Obj_Module_Arr[i].path);
            }
         }
    });
}
setImmediate(frida_Module,0);

输出如下：
模块名称: libhello.so
模块地址: 0xdf2d3000
大小: 24576
文件系统路径 /data/app/com.roysue.roysueapplication-7adQZoYIyp5t3G5Ef5wevQ==/lib/arm/libhello.so


```

这边如果去除判断的话会打印所有加载的`so`的信息，这里我们就知道了哪些方法返回了`Module`对象了，然后我们再继续深入学习`Module`对象自带的`API`。

### 1.5.5 enumerateImports()

该 API 会枚举模块中所有中的所有 Import 函数，示例代码如下。

```
function frida_Module() {
    Java.perform(function () {
        const hooks = Module.load('libhello.so');
        var Imports = hooks.enumerateImports();
        for(var i = 0; i < Imports.length; i++) {
            
            console.log("type:",Imports[i].type);
            
            console.log("name:",Imports[i].name);
            
            console.log("module:",Imports[i].module);
            
            console.log("address:",Imports[i].address);
         }
    });
}
setImmediate(frida_Module,0);

输出如下：
[Google Pixel::com.roysue.roysueapplication]-> type: function
name: __cxa_atexit
module: /system/lib/libc.so
address: 0xf58f4521
type: function
name: __cxa_finalize
module: /system/lib/libc.so
address: 0xf58f462d                                                                                                                                           
type: function
name: __stack_chk_fail
module: /system/lib/libc.so
address: 0xf58e2681
...


```

### 1.5.6 enumerateExports()

该 API 会枚举模块中所有中的所有`Export`函数，示例代码如下。

```
function frida_Module() {
    Java.perform(function () {
        const hooks = Module.load('libhello.so');
        var Exports = hooks.enumerateExports();
        for(var i = 0; i < Exports.length; i++) {
            
            console.log("type:",Exports[i].type);
            
            console.log("name:",Exports[i].name);
            
            console.log("address:",Exports[i].address);
         }
    });
}
setImmediate(frida_Module,0);

输出如下：
[Google Pixel::com.roysue.roysueapplication]-> type: function
name: Java_com_roysue_roysueapplication_hellojni_getSum
address: 0xdf2d411b
type: function
name: unw_save_vfp_as_X
address: 0xdf2d4c43
type: function
address: 0xdf2d4209
type: function
...


```

### 1.5.7 enumerateSymbols()

代码示例如下。

```
function frida_Module() {
    Java.perform(function () {
        const hooks = Module.load('libc.so');
        var Symbol = hooks.enumerateSymbols();
        for(var i = 0; i < Symbol.length; i++) {
            console.log("isGlobal:",Symbol[i].isGlobal);
            console.log("type:",Symbol[i].type);
            console.log("section:",JSON.stringify(Symbol[i].section));
            console.log("name:",Symbol[i].name);
            console.log("address:",Symbol[i].address);
         }
    });
}
setImmediate(frida_Module,0);

输出如下：
isGlobal: true
type: function
section: {"id":"13.text","protection":"r-x"}
name: _Unwind_GetRegionStart
address: 0xf591c798
isGlobal: true
type: function
section: {"id":"13.text","protection":"r-x"}
name: _Unwind_GetTextRelBase
address: 0xf591c7cc
...


```

### 1.5.8 Module.findExportByName(exportName), Module.getExportByName(exportName)

返回`so`文件中`Export`函数库中函数名称为`exportName`函数的绝对地址。

代码示例如下。

```
function frida_Module() {
    Java.perform(function () {
        Module.getExportByName('libhello.so', 'c_getStr')
        console.log("Java_com_roysue_roysueapplication_hellojni_getStr address:",Module.findExportByName('libhello.so', 'Java_com_roysue_roysueapplication_hellojni_getStr'));
        console.log("Java_com_roysue_roysueapplication_hellojni_getStr address:",Module.getExportByName('libhello.so', 'Java_com_roysue_roysueapplication_hellojni_getStr'));
    });
}
setImmediate(frida_Module,0);

输出如下：
Java_com_roysue_roysueapplication_hellojni_getStr address: 0xdf2d413d
Java_com_roysue_roysueapplication_hellojni_getStr address: 0xdf2d413d


```

### 1.5.9 Module.findBaseAddress(name)、Module.getBaseAddress(name)

返回`name`模块的基地址。

代码示例如下。

```
function frida_Module() {
    Java.perform(function () {
        var name = "libhello.so";
        console.log("so address:",Module.findBaseAddress(name));
        console.log("so address:",Module.getBaseAddress(name));
    });
}
setImmediate(frida_Module,0);

输出如下：
so address: 0xdf2d3000
so address: 0xdf2d3000


```

1.6 Memory 对象
-------------

`Memory`的一些`API`通常是对内存处理，譬如`Memory.copy()`复制内存，又如`writeByteArray`写入字节到指定内存中，那我们这章中就是学习使用`Memory API`向内存中写入数据、读取数据。

### 1.6.1 Memory.scan 搜索内存数据

其主要功能是搜索内存中以`address`地址开始，搜索长度为`size`，需要搜是条件是`pattern，callbacks`搜索之后的回调函数；此函数相当于搜索内存的功能。

我们来直接看例子，然后结合例子讲解，如下图`1-5`。

[![](https://p0.ssl.qhimg.com/t012b2f2c69c430786a.png)](https://p0.ssl.qhimg.com/t012b2f2c69c430786a.png)

图 1-5 IDA 中 so 文件某处数据

如果我想搜索在内存中`112A`地址的起始数据要怎么做，代码示例如下。

```
function frida_Memory() {
    Java.perform(function () {
        
        var module = Process.findModuleByName("libhello.so"); 
        
        var pattern = "03 49 ?? 50 20 44";
        
        console.log("base:"+module.base)
        
        var res = Memory.scan(module.base, module.size, pattern, {
            onMatch: function(address, size){
                
                console.log('搜索到 ' +pattern +" 地址是:"+ address.toString());  
            }, 
            onError: function(reason){
                
                console.log('搜索失败');
            },
            onComplete: function()
            {
                
                console.log("搜索完毕")
            }
          });
    });
}
setImmediate(frida_Memory,0);


```

先来看看回调函数的含义，`onMatch：function(address，size)`：使用包含作为`NativePointer`的实例地址的`address`和指定大小为数字的`size`调用，此函数可能会返回字符串`STOP`以提前取消内存扫描。`onError：Function(Reason)`：当扫描时出现内存访问错误时使用原因调用。`onComplete：function()`：当内存范围已完全扫描时调用。

我们来来说上面这段代码做了什么事情：搜索`libhello.so`文件在内存中的数据，搜索以`pattern`条件的在内存中能匹配的数据。搜索到之后根据回调函数返回数据。

我们来看看执行之后的效果图`1-6`。

[![](https://p1.ssl.qhimg.com/t0162e674a6690c141c.png)](https://p1.ssl.qhimg.com/t0162e674a6690c141c.png)

图 1-6 终端执行

我们要如何验证搜索到底是不是图`1-5`中`112A`地址，其实很简单。`so`的基址是`0xdf2d3000`，而搜到的地址是`0xdf2d412a`，我们只要`df2d412a-df2d3000=112A`。就是说我们已经搜索到了！

### 1.6.2 搜索内存数据 Memory.scanSync

功能与`Memory.scan`一样，只不过它是返回多个匹配到条件的数据。  
代码示例如下。

```
function frida_Memory() {
    Java.perform(function () {
        var module = Process.findModuleByName("libhello.so"); 
        var pattern = "03 49 ?? 50 20 44";
        var scanSync = Memory.scanSync(module.base, module.size, pattern);
        console.log("scanSync:"+JSON.stringify(scanSync));
    });
}
setImmediate(frida_Memory,0);

输出如下，可以看到地址搜索出来是一样的
scanSync:[{"address":"0xdf2d412a","size":6}]


```

### 1.6.3 内存分配 Memory.alloc

在目标进程中的堆上申请`size`大小的内存，并且会按照`Process.pageSize`对齐，返回一个`NativePointer`，并且申请的内存如果在`JavaScript`里面没有对这个内存的使用的时候会自动释放的。也就是说，如果你不想要这个内存被释放，你需要自己保存一份对这个内存块的引用。

使用案例如下

```
function frida_Memory() {
    Java.perform(function () {
        const r = Memory.alloc(10);
        console.log(hexdump(r, {
            offset: 0,
            length: 10,
            header: true,
            ansi: false
        }));
    });
}
setImmediate(frida_Memory,0);


```

以上代码在目标进程中申请了`10`字节的空间~ 我们来看执行脚本的效果图`1-7`。

[![](https://p2.ssl.qhimg.com/t01315da1bd55915455.png)](https://p2.ssl.qhimg.com/t01315da1bd55915455.png)

图 1-6 终端执行

可以看到在`0xdfe4cd40`处申请了`10`个字节内存空间~

也可以使用：  
`Memory.allocUtf8String(str)` 分配 utf 字符串  
`Memory.allocUtf16String` 分配 utf16 字符串  
`Memory.allocAnsiString` 分配 ansi 字符串

### 1.6.4 内存复制 Memory.copy

如同`c api memcp`一样调用，使用案例如下。

```
function frida_Memory() {
    Java.perform(function () {
        
        var module = Process.findModuleByName("libhello.so"); 
        
        var pattern = "03 49 ?? 50 20 44";
        
        var scanSync = Memory.scanSync(module.base, module.size, pattern);
        
        const r = Memory.alloc(10);
        
        Memory.copy(r,module.base,10);
        console.log(hexdump(r, {
            offset: 0,
            length: 10,
            header: true,
            ansi: false
        }));
    });
}
setImmediate(frida_Memory,0);


输出如下。
           0  1  2  3  4  5  6  7  8  9  A  B  C  D  E  F  0123456789ABCDEF
e8142070  7f 45 4c 46 01 01 01 00 00 00                    .ELF......


```

从`module.base`中复制`10`个字节的内存到新年申请的`r`内

### 1.6.6 写入内存 Memory.writeByteArray

将字节数组写入一个指定内存，代码示例如下:

```
function frida_Memory() {     
    Java.perform(function () {
        
        var arr = [ 0x72, 0x6F, 0x79, 0x73, 0x75, 0x65];
        
        const r = Memory.alloc(arr.length);
        
        Memory.writeByteArray(r,arr);
        
        console.log(hexdump(r, {
            offset: 0,
            length: arr.length,
            header: true,
            ansi: false
        }));  
    });
}
setImmediate(frida_Memory,0);


输出如下。
           0  1  2  3  4  5  6  7  8  9  A  B  C  D  E  F  0123456789ABCDEF
00000000  72 6f 79 73 75 65                                roysue


```

### 1.6.7 读取内存 Memory.readByteArray

将一个指定地址的数据，代码示例如下:

```
function frida_Memory() {     
    Java.perform(function () {
        
        var arr = [ 0x72, 0x6F, 0x79, 0x73, 0x75, 0x65];
        
        const r = Memory.alloc(arr.length);
        
        Memory.writeByteArray(r,arr);
        
        var buffer = Memory.readByteArray(r, arr.length);
        
        console.log("Memory.readByteArray:");
        console.log(hexdump(buffer, {
            offset: 0,
            length: arr.length,
            header: true,
            ansi: false
        }));
      });  
    });
}
setImmediate(frida_Memory,0);

输出如下。
[Google Pixel::com.roysue.roysueapplication]-> Memory.readByteArray:
           0  1  2  3  4  5  6  7  8  9  A  B  C  D  E  F  0123456789ABCDEF
00000000  72 6f 79 73 75 65                                roysue


```

结语
--

在这篇中我们学会了在 FRIDACLI 中如何输出想要输出格式，也学会如何声明变量，一步步的学习。在逐步的学习的过程，总是会遇到不同的问题。歌曲 <奇迹再现> 我相信你一定听过吧~，新的风暴已经出现, 怎么能够停止不前.. 遇到问题不要怕，总会解决的。

咱们会在下一篇中来学会如何使用 FRIDA 中的 Java 对象、Interceptor 对象、NativePointer 对象 NativeFunction 对象、NativeCallback 对象咱们拭目以待吧~