> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.anquanke.com](https://www.anquanke.com/post/id/195869)

[![](https://p1.ssl.qhimg.com/t0167bf11e82f16975b.png)](https://p1.ssl.qhimg.com/t0167bf11e82f16975b.png)

前言
--

上一篇我们学过了如何对 Java 层以及内存做处理，在这篇中我们来看看如何拦截 SO 层函数函数等等。

系列文章目录搬新 “家” 了，地址：[https://github.com/r0ysue/AndroidSecurityStudy](https://github.com/r0ysue/AndroidSecurityStudy) ，接下来窝会努力写更多喔 ~

注意，运行以下任何代码时都需要提前启动手机中的`frida-server`文件。

1.1 Java 对象
-----------

`Java`是十分哦不，应该说是极其重要的`API`，无论是想对`so`层亦或 java 层进行拦截，都必须编写`Java.perform`，在使用上面这些`API`时，应该都已经发现了吧~ 这章我们就来详细看看`Java`对象都有哪些`API`~

### 1.1.1 Java.available

该函数一般用来判断当前进程是否加载了`JavaVM，Dalvik`或`ART`虚拟机，咱们来看代码示例！

```
function frida_Java() {
    Java.perform(function () {
        
        if(Java.available)
        {
            
            console.log("hello java vm");
        }else{
            
            console.log("error");
        }
    });
}       
setImmediate(frida_Java,0);

输出如下。
hello java vm


```

核心注入的逻辑代码写在 <注入的逻辑代码> 内会非常的安全万无一失~

### 1.1.2 Java.androidVersion

显示 android 系统版本号

```
function frida_Java() {
    Java.perform(function () {
        
        if(Java.available)
        {
            
            console.log("",Java.androidVersion);
        }else{
            
            console.log("error");
        }
    });
}       
setImmediate(frida_Java,0);

输出如下。
9
因为我的系统版本是9版本~


```

### 1.1.3 枚举类 Java.enumerateLoadedClasses

该 API 枚举当前加载的所有类信息，它有一个回调函数分别是`onMatch、onComplete`函数，我们来看看代码示例以及效果！

```
function frida_Java() {
    Java.perform(function () {
        if(Java.available)
        {
            
            
            Java.enumerateLoadedClasses({
                
                onMatch: function (className)
                {
                    
                    console.log("",className);
                },
                
                onComplete: function ()
                {
                    
                    console.log("输出完毕");
                }
            });
        }else{
            console.log("error");
        }
    });
}       
setImmediate(frida_Java,0);


```

咱们来看执行的效果图 1-7。

[![](https://p1.ssl.qhimg.com/t01a8e923dd6208792f.png)](https://p1.ssl.qhimg.com/t01a8e923dd6208792f.png)

图 1-7 终端执行

它还有一个好兄弟 `Java.enumerateLoadedClassesSync()`，它返回的是一个数组。

### 1.1.4 枚举类加载器 Java.enumerateLoadedClasses

该`api`枚举`Java VM`中存在的类加载器，其有一个回调函数，分别是`onMatch: function (loader)`与`onComplete: function ()`，接着我们来看代码示例。

```
function frida_Java() {
    Java.perform(function () {
        if(Java.available)
        {
            
            Java.enumerateClassLoaders({
                
                onMatch: function (loader)
                {
                    console.log("",loader);
                },
                
                onComplete: function ()
                {
                    console.log("end");
                }
            });
        }else{
            console.log("error");
        }
    });
}       
setImmediate(frida_Java,0);


```

执行的效果图 1-8。

[![](https://p3.ssl.qhimg.com/t01205fe527c316b83e.png)](https://p3.ssl.qhimg.com/t01205fe527c316b83e.png)

图 1-8 终端执行

它也有一个好兄弟叫`Java.enumerateClassLoadersSync()`也是返回的数组。

### 1.1.5 附加调用 Java.perform

该`API`极其重要，`Java.perform（fn）`主要用于当前线程附加到`Java VM`并且调用`fn`方法。我们来看看示例代码及其含义。

```
function frida_Java() {
    
    Java.perform(function () {
        
        if(Java.available)
        {
            
            console.log("hello");
        }else{
            console.log("error");
        }
    });
}       
setImmediate(frida_Java,0);

输出如下。
[Google Pixel::com.roysue.roysueapplication]-> hello


```

没错你猜对了，它也有一个好兄弟。`Java.performNow(fn)~`

### 1.1.6 获取类 Java.use

`Java.use(className)，`动态获取`className`的类定义，通过对其调用`$new()`来调用构造函数，可以从中实例化对象。当想要回收类时可以调用`$Dispose()`方法显式释放，当然也可以等待`JavaScript`的垃圾回收机制，当实例化一个对象之后，可以通过其实例对象调用类中的静态或非静态的方法，官方代码示例定义如下。

```
Java.perform(function () {
  
  var Activity = Java.use('android.app.Activity');
  
  var Exception = Java.use('java.lang.Exception');
  
  Activity.onResume.implementation = function () {
    
    throw Exception.$new('Oh noes!');
  };
});


```

### 1.1.7 扫描实例类 Java.choose

在堆上查找实例化的对象，示例代码如下！

```
Java.perform(function () {
    
    Java.choose("android.view.View", {
        
        onMatch:function(instance){
            
            console.log(instance);
        },
        
        onComplete:function() {
            console.log("end")
        }});
});

输出如下：
android.view.View{2292774 V.ED..... ......ID 0,1794-1080,1920 #1020030 android:id/navigationBarBackground}
android.view.View{d43549d V.ED..... ......ID 0,0-1080,63 #102002f android:id/statusBarBackground}
end


```

### 1.1.8 类型转换器 Java.cast

`Java.cast(handle, klass)`，就是将指定变量或者数据强制转换成你所有需要的类型；创建一个 `JavaScript` 包装器，给定从 `Java.use（）` 返回的给定类`klas`的句柄的现有实例。此类包装器还具有用于获取其类的包装器的类属性，以及用于获取其类名的字符串表示的`$className`属性，通常在拦截`so`层时会使用此函数将`jstring、jarray`等等转换之后查看其值。

### 1.1.9 定义任意数组类型 Java.array

`frida`提供了在 js 代码中定义`java`数组的`api`，该数组可以用于传递给`java API`，我们来看看如何定义，代码示例如下。

```
Java.perform(function () {
        
        var intarr = Java.array('int', [ 1003, 1005, 1007 ]);
        
        var bytearr = Java.array('byte', [ 0x48, 0x65, 0x69 ]);
        for(var i=0;i<bytearr.length;i++)
        {
            
            console.log(bytearr[i])
        }
});


```

我们通过上面定义`int`数组和`byte`的例子可以知道其定义格式为`Java.array('type',[value1,value2,....]);`那它都支持`type`呢？我们来看看~

<table><thead><tr><th>索引</th><th>type</th><th>含义</th></tr></thead><tbody><tr><td>1</td><td>Z</td><td>boolean</td></tr><tr><td>2</td><td>B</td><td>byte</td></tr><tr><td>3</td><td>C</td><td>char</td></tr><tr><td>4</td><td>S</td><td>short</td></tr><tr><td>5</td><td>I</td><td>int</td></tr><tr><td>6</td><td>J</td><td>long</td></tr><tr><td>7</td><td>F</td><td>float</td></tr><tr><td>8</td><td>D</td><td>double</td></tr><tr><td>9</td><td>V</td><td>void</td></tr></tbody></table>

### 1.1.10 注册类 Java.registerClass(spec)

`Java.registerClass`：创建一个新的`Java`类并返回一个包装器，其中规范是一个包含：  
`name`：指定类名称的字符串。  
`superClass`：（可选）父类。要从 `java.lang.Objec`t 继承的省略。  
`implements`：（可选）由此类实现的接口数组。  
`fields`：（可选）对象，指定要公开的每个字段的名称和类型。  
`methods`：（可选）对象，指定要实现的方法。

注册一个类，返回类的实例，下面我贴一个基本的用法~ 实例化目标类对象并且调用类中的方法

```
Java.perform(function () {
          
          var hellojni = Java.registerClass({
            name: 'com.roysue.roysueapplication.hellojni'
          });
          console.log(hellojni.addInt(1,2));
});


```

我们再深入看看官方怎么来玩的：

```
var SomeBaseClass = Java.use('com.example.SomeBaseClass');

var X509TrustManager = Java.use('javax.net.ssl.X509TrustManager');

var MyWeirdTrustManager = Java.registerClass({
  
  name: 'com.example.MyWeirdTrustManager',
  
  superClass: SomeBaseClass,
  
  implements: [X509TrustManager],
  
  fields: {
    description: 'java.lang.String',
    limit: 'int',
  },
  
  methods: {
    
    $init: function () {
      console.log('Constructor called');
    },
    
    checkClientTrusted: function (chain, authType) {
      console.log('checkClientTrusted');
    },
    
    checkServerTrusted: [{
      
      returnType: 'void',
      
      argumentTypes: ['[Ljava.security.cert.X509Certificate;', 'java.lang.String'],
      
      implementation: function (chain, authType) {
         
        console.log('checkServerTrusted A');
      }
    }, {
      returnType: 'java.util.List',
      argumentTypes: ['[Ljava.security.cert.X509Certificate;', 'java.lang.String', 'java.lang.String'],
      implementation: function (chain, authType, host) {
        console.log('checkServerTrusted B');
        
        return null;
      }
    }],
    
    getAcceptedIssuers: function () {
      console.log('getAcceptedIssuers');
      return [];
    },
  }
});


```

我们来看看上面的示例都做了啥? 实现了证书类的`javax.net.ssl.X509TrustManager`类，，这里就是相当于自己在目标进程中重新创建了一个类，实现了自己想要实现的类构造，重构造了其中的三个接口函数、从而绕过证书校验。

### 1.1.11 Java.vm 对象

`Java.vm`对象十分常用，比如想要拿到`JNI`层的`JNIEnv`对象，可以使用`getEnv()`；我们来看看具体的使用和基本小实例。~

```
function frida_Java() {     
    Java.perform(function () {
         
         Interceptor.attach(Module.findExportByName("libhello.so" , "Java_com_roysue_roysueapplication_hellojni_getStr"), {
            onEnter: function(args) {
                console.log("getStr");
            },
            onLeave:function(retval){
                
                
                
                var env = Java.vm.getEnv();
                
                var jstring = env.newStringUtf('roysue');
                
                retval.replace(jstring);
                console.log("getSum方法返回值为:roysue")
            }
    });
}
setImmediate(frida_Java,0);


```

1.2 Interceptor 对象
------------------

该对象功能十分强大，函数原型是`Interceptor.attach(target, callbacks)`: 参数`target`是需要拦截的位置的函数地址，也就是填某个`so`层函数的地址即可对其拦截，`target`是一个`NativePointer`参数，用来指定你想要拦截的函数的地址，`NativePointer`我们也学过是一个指针。需要注意的是对于`Thumb`函数需要对函数地址`+1`，`callbacks`则是它的回调函数，分别是以下两个回调函数：

### 1.2.1 Interceptor.attach

`onEnter：`函数（`args`）：回调函数，给定一个参数`args`，可用于读取或写入参数作为 `NativePointer` 对象的数组。

`onLeave：`函数（`retval`）：回调函数给定一个参数 `retval`，该参数是包含原始返回值的 `NativePointer` 派生对象。可以调用 `retval.replace（1337）` 以整数 `1337` 替换返回值，或者调用 `retval.replace（ptr（"0x1234"））`以替换为指针。请注意，此对象在 `OnLeave` 调用中回收，因此不要将其存储在回调之外并使用它。如果需要存储包含的值，请制作深副本，例如：`ptr（retval.toString（））`。

我们来看看示例代码~

```
Interceptor.attach(Module.getExportByName('libc.so', 'read'), {
  
  onEnter: function (args) {
    this.fileDescriptor = args[0].toInt32();
  },
  
  onLeave: function (retval) {
    if (retval.toInt32() > 0) {
      
    }
  }
});


```

通过我们对`Interceptor.attach`函数有一些基本了解了~ 它还包含一些属性。

<table><thead><tr><th>索引</th><th>属性</th><th>含义</th></tr></thead><tbody><tr><td>1</td><td>returnAddress</td><td>返回地址，类型是<code>NativePointer</code></td></tr><tr><td>2</td><td>context</td><td>上下文：具有键<code>pc</code>和<code>sp</code>的对象，它们是分别为<code>ia32/x64/arm</code>指定<code>EIP/RIP/PC</code>和<code>ESP/RSP/SP的NativePointer</code>对象。其他处理器特定的键也可用，例如<code>eax、rax、r0、x0</code>等。也可以通过分配给这些键来更新寄存器值。</td></tr><tr><td>3</td><td>errno</td><td>当前<code>errno</code>值</td></tr><tr><td>4</td><td>lastError</td><td>当前操作系统错误值</td></tr><tr><td>5</td><td>threadId</td><td>操作系统线程 ID</td></tr><tr><td>6</td><td>depth</td><td>相对于其他调用的调用深度</td></tr></tbody></table>

我们来看看示例代码。

```
function frida_Interceptor() {
    Java.perform(function () {
        
        Interceptor.attach(Module.findExportByName("libhello.so" , "Java_com_roysue_roysueapplication_hellojni_getSum"), {
            onEnter: function(args) {
                
                console.log('Context information:');
                
                console.log('Context  : ' + JSON.stringify(this.context));
                
                console.log('Return   : ' + this.returnAddress);
                
                console.log('ThreadId : ' + this.threadId);
                console.log('Depth    : ' + this.depth);
                console.log('Errornr  : ' + this.err);
            },
            onLeave:function(retval){
            }
        });
    });
}
setImmediate(frida_Interceptor,0);


```

我们注入脚本之后来看看执行之后的效果以及输出的这些都是啥，执行的效果图`1-9`。

[![](https://p3.ssl.qhimg.com/t016ac097aac6fd0971.png)](https://p3.ssl.qhimg.com/t016ac097aac6fd0971.png)

图 1-9 终端执行

### 1.2.2 Interceptor.detachAll

简单来说这个的函数的作用就是让之前所有的`Interceptor.attach`附加拦截的回调函数失效。

### 1.2.3 Interceptor.replace

相当于替换掉原本的函数，用替换时的实现替换目标处的函数。如果想要完全或部分替换现有函数的实现，则通常使用此函数。，我们也看例子，例子是最直观的！代码如下。

```
function frida_Interceptor() {
    Java.perform(function () {
       
       
       var add_method = new NativeFunction(Module.findExportByName('libhello.so', 'c_getSum'), 
       'int',['int','int']);
       
       console.log("result:",add_method(1,2));
       
       Interceptor.replace(add_method, new NativeCallback(function (a, b) {
           
            return 123;
       }, 'int', ['int', 'int']));
       
       console.log("result:",add_method(1,2));
    });
}


```

我来看注入脚本之后的终端是是不是显示了`3`和`123`见下图`1-10`。

[![](https://p2.ssl.qhimg.com/t013fb138fe1c77d80c.png)](https://p2.ssl.qhimg.com/t013fb138fe1c77d80c.png)

图 1-10 终端执行

1.3 NativePointer 对象
--------------------

同等与 C 语言中的指针

### 1.3.1 new NativePointer(s)

声明定义 NativePointer 类型

```
function frida_NativePointer() {
    Java.perform(function () {
        
        const ptr1 = new NativePointer("100");
        console.log("ptr1:",ptr1);
        
        const ptr2 = new NativePointer("0x64");
        console.log("ptr2:",ptr2);        
        
        const ptr3 = new NativePointer(100);
        console.log("ptr3:",ptr3);
    });
}     
setImmediate(frida_NativePointer,0);

输出如下，都会自动转为十六进制的0x64
ptr1: 0x64
ptr2: 0x64
ptr3: 0x64


```

### 1.3.2 运算符以及指针读写 API

它也能调用以下运算符  
[![](https://p3.ssl.qhimg.com/t0135a7319c873d8d52.png)](https://p3.ssl.qhimg.com/t0135a7319c873d8d52.png)  
[![](https://p3.ssl.qhimg.com/t01821e14d1331f0aef.png)](https://p3.ssl.qhimg.com/t01821e14d1331f0aef.png)

看完 API 含义之后，我们来使用他们，下面该脚本是 readByteArray() 示例~

```
function frida_NativePointer() {
    Java.perform(function () {
       console.log("");
        
        var pointer = Process.findModuleByName("libc.so").base;
        
        console.log(pointer.readByteArray(0x10));
    });
}     
setImmediate(frida_NativePointer,0);
输出如下：
           0  1  2  3  4  5  6  7  8  9  A  B  C  D  E  F  0123456789ABCDEF
00000000  7f 45 4c 46 01 01 01 00 00 00 00 00 00 00 00 00  .ELF............


```

首先我先来用`readByteArray`函数来读取`libc.so`文件在内存中的数据，这样我们方便测试，我们从`libc`文件读取`0x10`个字节的长度，肯定会是`7F 45 4C 46...`因为`ELF`文件头部信息中的`Magic`属性。

### 1.3.3 readPointer()

咱们直接从`API`索引 11 开始玩 readPointer()，定义是从此内存位置读取`NativePointer`，示例代码如下。省略`function`以及`Java.perform`~

```
    var pointer = Process.findModuleByName("libc.so").base;
    console.log(pointer.readByteArray(0x10));
    console.log("readPointer():"+pointer.readPointer());
    输出如下。
    readPointer():0x464c457f


```

也就是将`readPointer`的前四个字节的内容转成地址产生一个新的`NativePointer`。

### 1.3.4 writePointer(ptr)

读取 ptr 指针地址到当前指针

```
        
        console.log("pointer :"+pointer);
        
        const r = Memory.alloc(4);
        
        r.writePointer(pointer);
        
        var buffer = Memory.readByteArray(r, 4);
        
        console.log(buffer);

输出如下。

pointer :0xf588f000

           0  1  2  3  4  5  6  7  8  9  A  B  C  D  E  F  0123456789ABCDEF
00000000  00 f0 88 f5                                      ....


```

### 1.3.5 readS32()、readU32()

从该内存位置读取有符号或无符号`8/16/32/etc`或浮点数 / 双精度值，并将其作为数字返回。这里拿`readS32()、readU32()`作为演示.

```
    
    console.log(pointer.readS32());
    
    console.log(pointer.readU32());


输出如下。
           0  1  2  3  4  5  6  7  8  9  A  B  C  D  E  F  0123456789ABCDEF
00000000  7f 45 4c 46 01 01 01 00 00 00 00 00 00 00 00 00  .ELF............
1179403647 == 0x464c457f
1179403647 == 0x464c457f


```

### 1.3.6 writeS32()、writeU32()

将有符号或无符号`8/16/32/`等或浮点数 / 双精度值写入此内存位置。

```
    
    const r = Memory.alloc(4);
    
    r.writeS32(0x12345678);
    
    console.log(r.readByteArray(0x10));

输出如下。
           0  1  2  3  4  5  6  7  8  9  A  B  C  D  E  F  0123456789ABCDEF
00000000  78 56 34 12 00 00 00 00 00 00 00 00 00 00 00 00  xV4.............


```

### 1.3.7 readByteArray(length))、writeByteArray(bytes)

`readByteArray(length))`连续读取内存`length`个字节，、`writeByteArray`连续写入内存`bytes`。

```
       
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

输出如下。       
Memory.readByteArray:
           0  1  2  3  4  5  6  7  8  9  A  B  C  D  E  F  0123456789ABCDEF
00000000  72 6f 79 73 75 65                                roysue


```

### 1.3.8 readCString([size = -1])、writeUtf8String(str)

`readCString`功能是读取指针地址位置的字节字符串，对应的`writeUtf8String`是写入指针地址位置的字符串处。（这里的`r`是接着上面的代码的变量）。

```
        
        console.log("readCString():"+r.readCString());
        
        const newPtrstr = r.writeUtf8String("haha");
        
        console.log("readCString():"+newPtrstr.readCString());


```

咱们来看看执行的效果~~ 见下图 1-11。

[![](https://p2.ssl.qhimg.com/t01e8013edc788aa585.png)](https://p2.ssl.qhimg.com/t01e8013edc788aa585.png)

图 1-11 终端执行

1.4 NativeFunction 对象
---------------------

创建新的`NativeFunction`以调用`address`处的函数 (用`NativePointer`指定)，其中`rereturn Type`指定返回类型，`argTypes`数组指定参数类型。如果不是系统默认值，还可以选择指定`ABI`。对于可变函数，添加一个‘.’固定参数和可变参数之间的`argTypes`条目，我们来看看官方的例子。

```
var friendlyFunctionName = new NativeFunction(friendlyFunctionPtr,
    'void', ['pointer', 'pointer']);

var returnValue = Memory.alloc(sizeOfLargeObject);

friendlyFunctionName(returnValue, thisPtr);


```

我来看看它的格式，函数定义格式为`new NativeFunction(address, returnType, argTypes[, options])，`参照这个格式能够创建函数并且调用`！returnType和argTypes[，]`分别可以填`void、pointer、int、uint、long、ulong、char、uchar、float、double、int8、uint8、int16、uint16、int32、uint32、int64、uint64`这些类型，根据函数的所需要的 type 来定义即可。

在定义的时候必须要将参数类型个数和参数类型以及返回值完全匹配，假设有三个参数都是`int`，则`new NativeFunction(address, returnType, ['int', 'int', 'int'])`，而返回值是`int`则`new NativeFunction(address, 'int', argTypes[, options])`，必须要全部匹配，并且第一个参数一定要是函数地址指针。

1.5 NativeCallback 对象
---------------------

`new NativeCallback(func，rereturn Type，argTypes[，ABI])：`创建一个由`JavaScript`函数`func`实现的新`NativeCallback`，其中`rereturn Type`指定返回类型，`argTypes`数组指定参数类型。您还可以指定`ABI`(如果不是系统默认值)。有关支持的类型和 Abis 的详细信息，请参见`NativeFunction`。注意，返回的对象也是一个`NativePointer`，因此可以传递给`Interceptor#replace`。当将产生的回调与`Interceptor.replace()`一起使用时，将调用 func，并将其绑定到具有一些有用属性的对象，就像`Interceptor.Attach()`中的那样。我们来看一个例子。如下，利用`NativeCallback`做一个函数替换。

```
    Java.perform(function () {
       var add_method = new NativeFunction(Module.findExportByName('libhello.so', 'c_getSum'), 
       'int',['int','int']);
       console.log("result:",add_method(1,2));
       
       Interceptor.replace(add_method, new NativeCallback(function (a, b) {
            return 123;
       }, 'int', ['int', 'int']));
       console.log("result:",add_method(1,2));
    });


```

结语
--

本篇咱们学习了非常实用的 API，如 Interceptor 对象对 so 层导出库函数拦截、NativePointer 对象的指针操作、NativeFunction 对象的实例化 so 函数的使用等都是当前灰常好用的函数~ 建议童鞋了多多尝试~