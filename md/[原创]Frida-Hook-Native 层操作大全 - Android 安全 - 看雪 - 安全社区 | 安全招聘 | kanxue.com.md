> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-280812.htm)

> [原创]Frida-Hook-Native 层操作大全

[原创]Frida-Hook-Native 层操作大全

11 小时前 192

### [原创]Frida-Hook-Native 层操作大全

 [![](http://passport.kanxue.com/upload/avatar/679/979679.png?1693204967)](user-home-979679.htm) [Shangwendada](user-home-979679.htm) ![](https://bbs.kanxue.com/view/img/rank/3.png)  ![](http://passport.kanxue.com/pc/view/img/star.gif) 11 小时前  192

前期准备
----

*   使用 jadx 进行逆向工程的基础知识。
*   能够理解 Java 代码。
*   能够编写简短的 JavaScript 代码片段。
*   熟悉 adb。
*   已 root 的设备。
*   对 x86/ARM64 汇编和逆向工程有基础了解。

1、Hook Native 层中调用的函数并且读取传入的参数
------------------------------

对于 Native 层的函数 Hook，我们使用如下模板

```
Interceptor.attach(targetAddress, {
    onEnter: function (args) {
        console.log('Entering ' + functionName);
        // Modify or log arguments if needed
    },
    onLeave: function (retval) {
        console.log('Leaving ' + functionName);
        // Modify or log return value if needed
    }
});

```

*   `Interceptor.attach`：将回调函数附加到指定的函数地址。`targetAddress` 应该是我们想要挂钩的本地函数的地址。
*   `onEnter`：当挂钩的函数被调用时，调用此回调。它提供对函数参数 (`args`) 的访问。
*   `onLeave`：当挂钩的函数即将退出时，调用此回调。它提供对返回值 (`retval`) 的访问。  
    需要获取 targetAddress 我们可以方便的使用如下 API

1.  `Module.enumerateExports()`  
    通过调用 Module.enumerateExports()，我们可以获取到导出函数的名称、地址以及其他相关信息。这些信息对于进行函数挂钩、函数跟踪或者调用其他函数都非常有用。
2.  `Module.getExportByName()`  
    当我们知道要查找的导出项的名称但不知道其地址时，可以使用 Module.getExportByName()。通过提供导出项的名称作为参数，这个函数会返回与该名称对应的导出项的地址。
3.  `Module.findExportByName()`  
    这与 Module.getExportByName() 是一样的。唯一的区别在于，如果未找到导出项，Module.getExportByName() 会引发异常，而 Module.findExportByName() 如果未找到导出项则返回 `null`。让我们看一个示例。
4.  `Module.getBaseAddress()`  
    通过调用 Module.getBaseAddress() 函数，我们可以获取指定模块的基址地址，然后可以基于这个基址地址进行偏移计算，以定位模块内部的特定函数、变量或者数据结构
5.  `Module.enumerateImports()`  
    通过调用 Module.enumerateImports() 函数，我们可以获取到指定模块导入的外部函数或变量的名称、地址以及其他相关信息。

### 例题 Frida-Labs 0x8

#### MainActivity

![](https://bbs.kanxue.com/upload/attach/202403/979679_7ERN84625EDTGHU.jpg)  
可以发现，程序从 EditText 控件中获取到了用户的输入，然后调用了 native 层中的 cmpstr 函数进行比较。

#### Navtive 层逻辑

![](https://bbs.kanxue.com/upload/attach/202403/979679_BH8PQCUN8R5W3D7.jpg)  
程序在 cmpstr 中使用了 strcmp 函数，那么我们只需要拿到 strcmp 函数的传入参数就可以知道程序的正确输入了

#### Hook begin

首先我们使用 Module.enumerateImports("libfrida0x8.so") 查看导入表  
![](https://bbs.kanxue.com/upload/attach/202403/979679_HXZ7QJ6H7U4X6KY.jpg)  
可以发现 strcmp 来自于 libc.so，那么我们就可以使用 Module.findExportByName("libc.so","strcmp"); 来获取 strcmp 的地址了  
![](https://bbs.kanxue.com/upload/attach/202403/979679_C7C5Q6RWU32M2CG.jpg)

获取了 strcmp 的地址就可以使用之前给的模板进行 Hook 了

```
function hook(){
 
    var targetAddress = Module.findExportByName("libc.so","strcmp");
    console.log("Strcmp Address: ",targetAddress.toString(16));
 
    Interceptor.attach(targetAddress,{
        onEnter:function (args){
             
        },onLeave:function(retval){
 
        }
    })
    console.log("success!");
}
 
 
function main(){
    Java.perform(function (){
        hook();
    })
}
setImmediate(main);

```

但是我们需要注意的是 strcmp 可能不止调用一次，因此我们需要判断 strcmp 的第一个参数是否为 0 我们才进行操作，不然 hook 可能会一直循环输出  
![](https://bbs.kanxue.com/upload/attach/202403/979679_Z2MVWBT4SNTBNHY.jpg)  
因此我们可以使用 Memory.readUtf8String(args[0]); 来获取我们的输入字符串，平且使用 if (input.includes("111")) 来判断

### hook 代码

```
function hook(){
 
    var targetAddress = Module.findExportByName("libc.so","strcmp");
    console.log("Strcmp Address: ",targetAddress.toString(16));
 
    Interceptor.attach(targetAddress,{
        onEnter:function (args){
            var input = Memory.readUtf8String(args[0]);
            if (input.includes("111")){
                console.log(Memory.readUtf8String(args[1]));
            }
 
        },onLeave:function(retval){
 
        }
    })
    console.log("success!");
}
 
 
function main(){
    Java.perform(function (){
        hook();
    })
}
setImmediate(main);

```

![](https://bbs.kanxue.com/upload/attach/202403/979679_RUPEFUFA7SG7DTQ.jpg)

2、Hook 修改 native 层程序返回值
-----------------------

首先还是给出 hook 的模板如下：

```
Interceptor.attach(functionaddr, {
    onEnter: function (args) {
 
    },
    onLeave: function (retval) {
 
    }
});

```

可以看到在 onLeave 中有一个参数 retval，这个 retval，就是我们 hook 上的程序的返回值，我们可以使用 retval.replace(val) 来修改返回值。

### 例题 Frida-labs 0x9

#### MainActivity

![](https://bbs.kanxue.com/upload/attach/202403/979679_6CTE2Q9DXGEQC5T.jpg)  
可以发现程序根据 native 层的 check_flag 方法的返回值

#### check_flag

![](https://bbs.kanxue.com/upload/attach/202403/979679_YUWA3KUKDV2NZ3C.jpg)  
只是简简单单的返回了一个 1

#### hookbegin

首先使用 Module.enumerateExports("liba0x9.so")，查看导出表，看看 check_flag 方法的偏移地址  
![](https://bbs.kanxue.com/upload/attach/202403/979679_JH9JUFK33JZA5ZA.jpg)  
然后就可以使用模板一把梭了

#### hook 代码

```
function hook(){
    var check_flag = Module.enumerateExports("liba0x9.so")[0]["address"];
    console.log("Func address = ",check_flag);
    Interceptor.attach(check_flag,{
        onEnter:function (args){
 
        },onLeave:function (retval){
            console.log("Origin retval : ",retval);
            retval.replace(1337);
        }
    })
}
function main(){
    Java.perform(function (){
        hook();
    })
}
setImmediate(hook);

```

![](https://bbs.kanxue.com/upload/attach/202403/979679_SU54WWPKETXWTT7.jpg)

3、调用 native 层中未被调用的方法
---------------------

让我提供一个模板。

```
var native_adr = new NativePointer();
const native_function = new NativeFunction(native_adr, '', ['argument_data_type']);
native_function(); 
```

让我逐行解释。

```
var native_adr = new NativePointer(); 
```

要在 Frida 中调用一个本地函数，我们需要一个 `NativePointer` 对象。我们应该将要调用的本地函数的地址传递给 `NativePointer` 构造函数。接下来，我们将创建 `NativeFunction` 对象，它表示我们想要调用的实际本地函数。它在本地函数周围创建一个 JavaScript 包装器，允许我们从 Frida 调用该本地函数。

```
const native_function = new NativeFunction(native_adr, '', ['argument_data_type']); 
```

第一个参数应该是 `NativePointer` 对象，第二个参数是本地函数的返回类型，第三个参数是要传递给本地函数的参数的数据类型列表。现在我们可以像在 Java 空间中那样调用该方法了。

```
native_function(); 
```

好的，我们明白了。让我们来看看例题。

### 例题 Frida-labs 0xA

#### MainActivity

![](https://bbs.kanxue.com/upload/attach/202403/979679_5RUYY7RXQTT68U4.jpg)  
发现就是在主函数中加载了 stringFromJNI

#### native

![](https://bbs.kanxue.com/upload/attach/202403/979679_TB36MTWACDWB86E.jpg)  
没有关于 flag 的信息，但是有未被调用的 flag 函数，我们直接使用 hook 调用它输出 log  
![](https://bbs.kanxue.com/upload/attach/202403/979679_QPVUWYSEUUAPMWD.jpg)

#### hook 代码

```
function hook(){
    var a = Module.findBaseAddress("libfrida0xa.so");
    var b = Module.enumerateExports("libfrida0xa.so");
    var get_flagaddress = null;
    var mvaddress = null;
    for(var i = 0 ; b[i]!= null ; i ++ ){
        // console.log(b[i]["name"])
        if(b[i]["name"] == "_Z8get_flagii"){
            console.log("function get_flag : ",b[i]["address"]);
            console.log((b[i]["address"] - a).toString(16));
     //       mvaddress = b[i]["address"] - a;
            get_flagaddress = b[i]["address"];
        }
    }
    console.log(ptr.toString(16));
 
    var get_flag_ptr = new NativePointer(get_flagaddress);
    const get_flag = new NativeFunction(get_flag_ptr,'char',['int','int']);
    var flag = get_flag(1,2);
    console.log(flag)
    //console.log(b);
}
 
function main(){
    Java.perform(function (){
        hook();
    })
}
setImmediate(main)

```

![](https://bbs.kanxue.com/upload/attach/202403/979679_PWAF44KUQZEUY9K.jpg)

4、更改 Native 层方法的汇编指令
--------------------

首先我们先看来自 x86 指令集的 frida 使用模板

```
var writer = new X86Writer(opcodeaddr);
Memory.protect(opcodeaddr, 0x1000, "rwx");
try {
 
  writer.flush();
 
} finally {
 
  writer.dispose();
}

```

**`X86Writer`的实例化：**

*   `var writer = new X86Writer(<指令的地址>);`
*   这将创建一个`X86Writer`类的实例，并指定我们要修改的指令的地址。这设置了写入器以操作指定的内存位置。

** 插入指令 **

*   `try { /* 在此处插入指令 */ }`
*   在`try`块内，我们可以插入要修改 / 添加的 x86 指令。`X86Writer`实例提供了各种方法来插入各种 x86 指令。我们可以查阅文档以了解详情。

**刷新更改：**

*   `writer.flush();`
*   插入指令后，调用`flush`方法将更改应用到内存中。这确保修改后的指令被写入内存位置。

**清理：**

*   `finally { /* 释放X86Writer以释放资源 */ writer.dispose(); }`
*   `finally`块用于确保`X86Writer`资源得到适当清理。调用`dispose`方法释放与`X86Writer`实例关联的资源。

**解除段只读权限**  
`Memory.protect` 。我们可以使用这个函数来修改内存区域的保护属性。`Memory.protect` 函数的语法如下：

```
Memory.protect(地址, 大小, 保护属性);

```

*   `地址`：要更改保护的内存区域的起始地址。
*   `大小`：内存区域的大小，以字节为单位。
*   `保护属性`：内存区域的保护属性。

那么如何使用进行覆写呢  
对于 x86 系统而言我们首先需要查看官方文档中的使用方法  
[https://frida.re/docs/javascript-api/#x86writer](https://frida.re/docs/javascript-api/#x86writer)

![](https://bbs.kanxue.com/upload/attach/202403/979679_TNMKTACGGURKN98.jpg)  
对与 arm64 系统而言，我们使用如下 api  
[https://frida.re/docs/javascript-api/#arm64writer](https://frida.re/docs/javascript-api/#arm64writer)

![](https://bbs.kanxue.com/upload/attach/202403/979679_S3PF4D68H3739VJ.jpg)  
接下来让我用一个用例程序来讲一下这个指令的用法，我们示范的内容为 arm64 架构

### 例题 Frida-labs 0xB

#### MainActivity

首先我们看到 MainActivity 函数内容  
![](https://bbs.kanxue.com/upload/attach/202403/979679_U7CAKDKWMUP47N9.jpg)  
发现 MainActivity 就是在用户点击按钮后调用了 getflag 方法，但是正常点击 getflag 方法并不会返回 flag 值。

#### Native 层内容

![](https://bbs.kanxue.com/upload/attach/202403/979679_P2HKZNEMUW8ZWFZ.jpg)  
惊讶的发现 MainActivity 中什么都没有，显然这是不存在的。接下来我们到控制流窗口中查看。  
![](https://bbs.kanxue.com/upload/attach/202403/979679_UNCEPTUBWVVGVW9.jpg)  
查看控制流发现程序出现了永假条件跳转。导致导致 ida 识别不到输出 flag 的功能。那么我们可以把这个 B.NE 给 Nop 掉即可  
首先我们需要计算 B.NE 的偏移地址  
![](https://bbs.kanxue.com/upload/attach/202403/979679_S5XNR4C4HA7X7NF.jpg)  
可以发现就是基地址增加 15248，然后我们覆写为 Nop 就可以了

#### Hook 代码

```
function hook(){
    var Base =  Module.getBaseAddress("libfrida0xb.so");
    console.log("Base address : ",Base);
    var BNE = Base.add(0x15248);
    Memory.protect(Base,0x1000,"rwx");
    var writer = new Arm64Writer(BNE);
    try{
        writer.putNop();
        writer.flush();
        console.log("Success!!");
    }finally {
        writer.dispose();
    }
}
 
function main(){
    Java.perform(function (){
        hook();
    })
}
 
setTimeout(main,1000);

```

![](https://bbs.kanxue.com/upload/attach/202403/979679_ZA3NWJCEBUK5DRY.jpg)

  

[[CTF 入门培训] 顶尖高校博士及硕士团队亲授《30 小时教你玩转 CTF》，视频 + 靶场 + 题目！助力进入 CTF 世界](http://www.kanxue.com/book-brief-170.htm#h3a6WRhDT9Q_3D)

[#HOOK 注入](forum-161-1-125.htm) [#逆向分析](forum-161-1-118.htm) [#工具脚本](forum-161-1-128.htm)